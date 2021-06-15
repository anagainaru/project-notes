# Brute force algorithm

## Description

Find collisions (based on predicates) between primitives in a brute force manner (check all the all).

<img width="868" alt="summary" src="https://user-images.githubusercontent.com/16229479/121951103-7c233480-cd28-11eb-847e-20aedbc16645.png">

Each tile of primitives / predicates contains multiple entries (to fill half of the scratch space). Assuming rank k contains `k_N` primitives and `k_M` predicates, rank k will check `k_N * k_M` combinations of primitives and predicates.

## Code

```c++
  template <class ExecutionSpace, class Primitives, class Predicates,
            class Callback>
  static void query(ExecutionSpace const &space, Primitives const &primitives,
                    Predicates const &predicates, Callback const &callback)
  {
    using TeamPolicy = Kokkos::TeamPolicy<ExecutionSpace>;
    using AccessPrimitives = AccessTraits<Primitives, PrimitivesTag>;
    using AccessPredicates = AccessTraits<Predicates, PredicatesTag>;
    using PredicateType = typename AccessTraitsHelper<AccessPredicates>::type;
    using PrimitiveType = typename AccessTraitsHelper<AccessPrimitives>::type;

    int const n_primitives = AccessPrimitives::size(primitives);
    int const n_predicates = AccessPredicates::size(predicates);
    int max_scratch_size = TeamPolicy::scratch_size_max(0) / 2;
    int const predicates_per_team = max_scratch_size / sizeof(PredicateType);
    int const primitives_per_team = max_scratch_size / sizeof(PrimitiveType);

    int const n_primitive_tiles =
        ceil((float)n_primitives / primitives_per_team);
    int const n_predicate_tiles = ceil((float)n_predicates / predicates_per_team);
    int const n_teams = n_primitive_tiles * n_predicate_tiles;

    using ScratchPredicateType =
        Kokkos::View<PredicateType *,
                     typename ExecutionSpace::scratch_memory_space,
                     Kokkos::MemoryTraits<Kokkos::Unmanaged>>;
    using ScratchPrimitiveType =
        Kokkos::View<PrimitiveType *,
                     typename ExecutionSpace::scratch_memory_space,
                     Kokkos::MemoryTraits<Kokkos::Unmanaged>>;
    int scratch_size = ScratchPredicateType::shmem_size(predicates_per_team) +
                   ScratchPrimitiveType::shmem_size(primitives_per_team);

    Kokkos::parallel_for(
        "ArborX::BruteForce::query::spatial::"
        "check_all_predicates_against_all_primitives",
        TeamPolicy((long)n_teams, Kokkos::AUTO)
            .set_scratch_size(0, Kokkos::PerTeam(scratch_size)),
        KOKKOS_LAMBDA(const typename TeamPolicy::member_type &teamMember) {
          // select the tile of predicates/primitives checked by each team
          int predicate_start = predicates_per_team *
                (teamMember.league_rank() / n_primitive_tiles);
          int primitive_start = primitives_per_team *
                (teamMember.league_rank() % n_primitive_tiles);

          int predicates_in_this_team = KokkosExt::min(
          predicates_per_team, n_predicates - predicate_start);
          int primitives_in_this_team = KokkosExt::min(
              primitives_per_team, n_primitives - primitive_start);

          ScratchPredicateType scratch_predicates(teamMember.team_scratch(0),
                                                  predicates_per_team);
          ScratchPrimitiveType scratch_primitives(teamMember.team_scratch(0),
                                                  primitives_per_team);
          // rank 0 in each team fills the scratch space with the
          // predicates / primitives in the tile
          if (teamMember.team_rank() == 0)
          {
            Kokkos::parallel_for(
                    Kokkos::ThreadVectorRange(
                        teamMember, (long)predicates_in_this_team),
                    [&](const int q) {
                      scratch_predicates(q) = AccessPredicates::get(
                              predicates, predicate_start + q);
                    });
            Kokkos::parallel_for(
                    Kokkos::ThreadVectorRange(
                        teamMember, (long)primitives_in_this_team),
                    [&](const int j) {
                      scratch_primitives(j) = AccessPrimitives::get(
                              primitives, primitive_start + j);
                    });
          }
          teamMember.team_barrier();

          // start threads for every predicate / primitive combination
          Kokkos::parallel_for(
              Kokkos::TeamThreadRange(
                  teamMember, (long)primitives_in_this_team),
              [&](const int j) {
                Kokkos::parallel_for(
                    Kokkos::ThreadVectorRange(
                        teamMember, predicates_in_this_team),
                    [&](const int q) {
                      auto const &predicate = scratch_predicates(q);
                      auto const &primitive = scratch_primitives(j);
                      if (predicate(primitive))
                      {
                        callback(predicate, j + primitive_start);
                      }
                    });
              });
        });
```

## Testing
```bash
$ export DYLD_LIBRARY_PATH=/Users/95j/work/arborx/boost_1_75_0/install/lib
$ ctest
Test project /Users/95j/work/arborx/ArborX/build
      Start  1: ArborX_DetailsUtils_Test
 1/15 Test  #1: ArborX_DetailsUtils_Test ...................   Passed    0.01 sec
      Start  2: ArborX_Geometry_Test
 2/15 Test  #2: ArborX_Geometry_Test .......................   Passed    0.01 sec
      Start  3: ArborX_QueryTree_Test
 3/15 Test  #3: ArborX_QueryTree_Test ......................   Passed    0.27 sec
      Start  4: ArborX_DetailsTreeConstruction_Test
 4/15 Test  #4: ArborX_DetailsTreeConstruction_Test ........   Passed    0.01 sec
      Start  5: ArborX_DetailsContainers_Test
 5/15 Test  #5: ArborX_DetailsContainers_Test ..............   Passed    0.01 sec
      Start  6: ArborX_DetailsTreeTraversal_Test
 6/15 Test  #6: ArborX_DetailsTreeTraversal_Test ...........   Passed    0.03 sec
      Start  7: ArborX_DetailsBatchedQueries_Test
 7/15 Test  #7: ArborX_DetailsBatchedQueries_Test ..........   Passed    0.01 sec
      Start  8: ArborX_DetailsCrsGraphWrapperImpl_Test
 8/15 Test  #8: ArborX_DetailsCrsGraphWrapperImpl_Test .....   Passed    0.01 sec
      Start  9: ArborX_BoostAdapters_Test
 9/15 Test  #9: ArborX_BoostAdapters_Test ..................   Passed    0.01 sec
      Start 10: ArborX_HostAccessTraits_Example
10/15 Test #10: ArborX_HostAccessTraits_Example ............   Passed    0.01 sec
      Start 11: ArborX_Callback_Example
11/15 Test #11: ArborX_Callback_Example ....................   Passed    0.01 sec
      Start 12: ArborX_BruteForce_Example
12/15 Test #12: ArborX_BruteForce_Example ..................   Passed    0.11 sec
      Start 13: ArborX_DBSCAN
13/15 Test #13: ArborX_DBSCAN ..............................   Passed    0.01 sec
      Start 14: ArborX_RayTracing_Example
14/15 Test #14: ArborX_RayTracing_Example ..................   Passed    0.05 sec
      Start 15: ArborX_BoundingVolumeHierarchy_Benchmark
15/15 Test #15: ArborX_BoundingVolumeHierarchy_Benchmark ...   Passed    5.52 sec

100% tests passed, 0 tests failed out of 15

Total Test time (real) =   6.07 sec
```
## Performance

Performance results in the dedicated [performance page](./performance.md)
