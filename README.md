import { interval, of, pipe, startWith, switchMap, tap, map, Observable } from 'rxjs';

import { computed, inject } from '@angular/core';
import { tapResponse } from '@ngrx/operators';
import { patchState, signalStore, withComputed, withHooks, withMethods, withState } from '@ngrx/signals';
import { rxMethod } from '@ngrx/signals/rxjs-interop';
import { Tester, TesterWithLocation } from '@shared/interfaces';
import {
    LayoutDTO,
    SiteYieldCollectionDTO,
    TesterDetailsDTO,
    TesterStatusDTO,
    TDTimesDTO,
} from '@swagger/model/models';
import { ControlRoomApi } from '@swagger/api/controlRoom.service';

import { filterItemsByQuery } from '@shared/utils';
import { TestersApi } from '@swagger/api/testers.service';
import { HttpErrorResponse } from '@angular/common/http';
import { TOOLBAR_STORE } from '../toolbar-store/toolbar.store';
import {
    calculateMetrics,
    getSelectedTester,
    getTestersChangeUpdates,
    mapTesterData,
} from './testers-selectors.helper';
import { TestersState } from './testers.interface';

const initialState: TestersState = {
    testers: [],
    testerLocations: [],
    imageZoom: 100,
    selectedTesterID: null,
    isLoading: false,
    intervalSubscription: null,
    testersChangeTrackingDictionary: null,
    siteYieldData: null,
    touchDownTimesData: [],
};

export const TESTERS_STORE = signalStore(
    { providedIn: 'root' },
    withState(initialState),
    withComputed(({ testers, selectedTesterID, isLoading, testerLocations }, toolbarStore = inject(TOOLBAR_STORE)) => {
        const filteredTesters = computed(() => filterItemsByQuery(toolbarStore.activeFilter(), testers()));

        return {
            filteredTesters,
            testersData: computed(() => mapTesterData(filteredTesters())),
            metrics: computed(() => calculateMetrics(filteredTesters())),
            isDataLoading: computed(() => toolbarStore.isLoading() || isLoading()),
            isToolbarLoading: computed(() => toolbarStore.isLoading()),
            testerWithLocations: computed((): TesterWithLocation[] => {
                return filteredTesters().map((tester): TesterWithLocation => {
                    const location = testerLocations().find(loc => loc.testerId === tester.testerId);

                    return {
                        ...tester,
                        x: location?.x ?? 0,
                        y: location?.y ?? 0,
                    };
                });
            }),
            selectedTester: computed(() => {
                if (selectedTesterID()) {
                    return getSelectedTester(testers(), selectedTesterID()!);
                }

                return undefined;
            }),
        };
    }),
    withMethods(
        (
            state,
            toolbarStore = inject(TOOLBAR_STORE),
            controlRoomApi = inject(ControlRoomApi),
            testersApi = inject(TestersApi),
        ) => {
            return {
                getLayoutTesters: rxMethod<LayoutDTO | null>(
                    pipe(
                        switchMap(layout => {
                            const layoutId = layout?.id;

                            if (!layoutId) {
                                return of([]);
                            }

                            patchState(state, { isLoading: true });

                            return controlRoomApi.apiV1ControlRoomLayoutsLayoutIdGet(layoutId).pipe(
                                tapResponse({
                                    next: ({ testerLocations, imageZoom, name, id }) => {
                                        const layouts = toolbarStore.layouts();
                                        const currentLayout = layouts.find(l => l.id === id);

                                        // Check if layout was renamed (same ID, different name)
                                        if (currentLayout && currentLayout.name !== name) {
                                            toolbarStore.setBannerMessage({
                                                title: $localize`Layout Renamed`,
                                                description: $localize`This layout was renamed from ${currentLayout.name} to ${name}.`,
                                            });

                                            // Fetch layouts to get updated list
                                            toolbarStore.getLayouts();
                                        }

                                        patchState(state, {
                                            testerLocations: testerLocations ?? [],
                                            imageZoom: imageZoom ?? 100,
                                        });
                                    },
                                    error: (error: HttpErrorResponse) => {
                                        // Check if layout was deleted (id is on the layout list but response is 404)
                                        // It refetch the layouts to get the updated list and set default layout as selected
                                        if (error.status === 404) {
                                            const layouts = toolbarStore.layouts();
                                            const deletedLayout = layouts.find(l => l.id === layoutId);

                                            toolbarStore.getLayouts(deletedLayout);
                                        }
                                        patchState(state, { testerLocations: [] });
                                    },
                                    finalize: () => patchState(state, { isLoading: false }),
                                }),
                            );
                        }),
                    ),
                ),
                getTesters: rxMethod<TesterDetailsDTO[]>(
                    pipe(
                        switchMap(locations => {
                            const layoutId = toolbarStore.selectedLayout()?.id;
                            const testerIds = locations.map(loc => loc.testerId);

                            if (!layoutId || !testerIds.length) {
                                return of([]);
                            }

                            const TESTERS_POLLING_INTERVAL_MS = 5000;

                            const requst: Observable<Tester[]> = controlRoomApi
                                .apiV1ControlRoomLayoutsLayoutIdTesterStatesGet(layoutId, undefined, undefined)
                                .pipe(map((testerStatuses: TesterStatusDTO[]) => testerStatuses as Tester[]));

                            return interval(TESTERS_POLLING_INTERVAL_MS).pipe(
                                startWith(0),
                                tap(() => patchState(state, { isLoading: true })),
                                switchMap(() =>
                                    requst.pipe(
                                        tapResponse({
                                            next: (testers: Tester[]) => {
                                                const testersChangeTrackingDictionary = getTestersChangeUpdates(
                                                    testers,
                                                    state.testersChangeTrackingDictionary(),
                                                );

                                                // Update both states together
                                                patchState(state, {
                                                    testers,
                                                    testersChangeTrackingDictionary,
                                                });
                                            },
                                            error: () => patchState(state, { testers: [] }),
                                            finalize: () => patchState(state, { isLoading: false }),
                                        }),
                                    ),
                                ),
                            );
                        }),
                    ),
                ),
                setSelectedTesterId(testerId: string | null): void {
                    patchState(state, { selectedTesterID: testerId });
                },
                getTesterById(testerId: string): Tester | undefined {
                    return state.testers().find(tester => tester.testerId.toString() === testerId.toString());
                },
                getSiteYieldData: rxMethod<number>(
                    pipe(
                        switchMap(testerId => {
                            if (!testerId) {
                                return of(null);
                            }

                            return testersApi.apiV1TestersTesterIdRunsRunIdSiteYieldGet(testerId, 1).pipe(
                                tapResponse({
                                    next: (siteYieldData: SiteYieldCollectionDTO) =>
                                        patchState(state, { siteYieldData }),
                                    error: () => patchState(state, { siteYieldData: null }),
                                }),
                            );
                        }),
                    ),
                ),
                clearSiteYieldData: (): void => {
                    patchState(state, { siteYieldData: null });
                },
                fetchTouchDownTimes: rxMethod<{ testerId: number; fromTD: number | undefined }>(
                    pipe(
                        switchMap(({ testerId, fromTD }) => {
                            const runId = 1; // Mocked until runs are implemented on backend
                            if (!testerId) {
                                return of(null);
                            }

                            return testersApi.apiV1TestersTesterIdRunsRunIdTdtimesGet(testerId, runId, fromTD).pipe(
                                tapResponse({
                                    next: (newTDTimes: TDTimesDTO[]) =>
                                        patchState(state, {
                                            touchDownTimesData: [...state.touchDownTimesData(), ...newTDTimes],
                                        }),
                                    error: () =>
                                        patchState(state, { touchDownTimesData: [...state.touchDownTimesData()] }),
                                }),
                            );
                        }),
                    ),
                ),
                clearTouchDownTimesData: (): void => {
                    patchState(state, { touchDownTimesData: [] });
                },
            };
        },
    ),
    withHooks({
        onInit({ getTesters, getLayoutTesters, testerLocations }, toolbarStore = inject(TOOLBAR_STORE)) {
            const layout = toolbarStore.selectedLayout;

            getLayoutTesters(layout);
            getTesters(testerLocations);
        },



        interface.ts file:
        import { Subscription } from 'rxjs';

import { SelectorOption, Tester } from '@shared/interfaces';
import { SiteYieldCollectionDTO, TDTimesDTO, TesterDetailsDTO } from '@swagger/model/models';

import type { TableRecord } from '@ni/nimble-angular/table';

export interface TestersState {
    testers: Tester[];
    selectedTesterID: string | null;
    isLoading: boolean;
    intervalSubscription: Subscription | null;
    testerLocations: TesterDetailsDTO[];
    imageZoom: number;
    testersChangeTrackingDictionary: TestersChangeTracking | null;
    siteYieldData: SiteYieldCollectionDTO | null;
    touchDownTimesData: TDTimesDTO[];
}

export interface MetricItem extends SelectorOption {
    value: number;
}

// Use Omit to exclude 'state' and 'elapsedTimeSecs' from Tester, then add any needed fields
export interface TesterData extends Omit<Tester, 'state' | 'elapsedTimeSecs'>, TableRecord {
    id: string;
    yieldValue: number;
    elapsedTimeSecs: string;
}

export interface TestersChangeTracking {
    [key: string]: TesterChangeTracking;
}

export interface TesterChangeTracking {
    startTime: string;
    slot: number | null;
    isChanged: boolean;
}





helper.ts file:
import { Tester } from '@shared/interfaces';

import { MetricItem, TesterData, TestersChangeTracking } from './testers.interface';

function formatElapsedTime(seconds: number | null): string {
    if (!seconds) {
        return '-';
    }

    const hours = Math.floor(seconds / 3600);
    const minutes = Math.floor((seconds % 3600) / 60);
    const parts = [hours && `${hours}h`, minutes && `${minutes}m`].filter(Boolean);

    return parts.length ? parts.join(' ') : `${seconds}s`;
}

export function mapTesterData(testers: Tester[]): TesterData[] {
    return testers.map(({ yield: yieldValue, elapsedTimeSecs, ...rest }) => {
        const mappedRest = Object.entries(rest).reduce<{ [key: string]: unknown }>((acc, [key, value]) => {
            acc[key] = value ?? '-';
            return acc;
        }, {});

        return {
            ...mappedRest,
            id: rest.testerId.toString(),
            yieldValue: yieldValue ? Number(yieldValue.toFixed(1)) : yieldValue,
            elapsedTimeSecs: formatElapsedTime(elapsedTimeSecs ?? null),
        } as TesterData;
    });
}

export function calculateMetrics(testers: Tester[]): MetricItem[] {
    const TRACKED_STATUSES = ['Standby', 'Productive', 'Unknown', 'Pause', 'Engineering'] as const;

    const counts = TRACKED_STATUSES.reduce<{ [k: string]: number }>((acc, status) => {
        acc[status] = 0;
        return acc;
    }, {});

    testers.forEach(({ status }) => {
        if (status !== undefined && status in counts) {
            counts[status] += 1;
        }
    });

    return TRACKED_STATUSES.map(status => ({
        label: status,
        value: counts[status],
    }));
}

export function getSelectedTester(testers: Tester[], selectedTesterID: string): Tester | undefined {
    return testers.find((tester: Tester) => tester.testerId.toString() === selectedTesterID);
}

export function getTestersChangeUpdates(testers: Tester[], testersStartTime: TestersChangeTracking | null): TestersChangeTracking {
    const updatedTestersStartTime: TestersChangeTracking = { ...testersStartTime };

    testers.forEach((tester: Tester) => {
        if (tester.startTime) {
            const existingEntry = testersStartTime?.[tester.testerId];
            const isValueChanged = (existingEntry?.startTime !== tester.startTime || existingEntry.slot !== tester.slot);
            const isChanged = !!((!existingEntry || isValueChanged));
            updatedTestersStartTime[tester.testerId] = { startTime: tester.startTime, slot: tester.slot ?? null, isChanged };
        }
    });

    return updatedTestersStartTime;
}

    }),
);
