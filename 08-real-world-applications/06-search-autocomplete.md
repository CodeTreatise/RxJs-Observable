# Search & Autocomplete Patterns

## Learning Objectives
By the end of this lesson, you will be able to:
- Build sophisticated search experiences with debouncing and caching
- Implement intelligent autocomplete with ranking and filtering
- Create search result highlighting and navigation
- Handle search history and saved searches
- Implement faceted search with multiple filters
- Build search analytics and performance optimization
- Create advanced search patterns like typeahead and instant search
- Handle search state management and URL synchronization

## Table of Contents
1. [Search Architecture & Core Patterns](#search-architecture--core-patterns)
2. [Advanced Autocomplete Service](#advanced-autocomplete-service)
3. [Search State Management](#search-state-management)
4. [Faceted Search & Filtering](#faceted-search--filtering)
5. [Search History & Persistence](#search-history--persistence)
6. [Search Results & Highlighting](#search-results--highlighting)
7. [Performance Optimization](#performance-optimization)
8. [Advanced Search Patterns](#advanced-search-patterns)

## Search Architecture & Core Patterns

### Core Search Service

```typescript
// search.service.ts
import { Injectable } from '@angular/core';
import { 
  BehaviorSubject, 
  Observable, 
  Subject, 
  combineLatest,
  merge,
  timer,
  of,
  throwError,
  EMPTY
} from 'rxjs';
import { 
  debounceTime, 
  distinctUntilChanged, 
  switchMap,
  map,
  tap,
  filter,
  take,
  takeUntil,
  startWith,
  catchError,
  shareReplay,
  scan,
  withLatestFrom
} from 'rxjs/operators';

export interface SearchQuery {
  term: string;
  filters: SearchFilters;
  sort: SortOption;
  pagination: PaginationOptions;
  facets?: string[];
}

export interface SearchFilters {
  [key: string]: any;
  category?: string[];
  dateRange?: DateRange;
  priceRange?: PriceRange;
  location?: LocationFilter;
  tags?: string[];
}

export interface SearchResult<T = any> {
  items: T[];
  totalCount: number;
  facets: SearchFacet[];
  suggestions: string[];
  searchTime: number;
  query: SearchQuery;
  hasMore: boolean;
}

export interface SearchFacet {
  name: string;
  values: FacetValue[];
  type: 'checkbox' | 'range' | 'select' | 'autocomplete';
}

export interface FacetValue {
  value: string;
  count: number;
  selected: boolean;
}

export interface SortOption {
  field: string;
  direction: 'asc' | 'desc';
}

export interface PaginationOptions {
  page: number;
  size: number;
}

export interface SearchState {
  query: SearchQuery;
  results: SearchResult | null;
  isLoading: boolean;
  error: string | null;
  lastSearchTime: Date | null;
}

@Injectable({
  providedIn: 'root'
})
export class SearchService {
  private searchStateSubject = new BehaviorSubject<SearchState>(this.getInitialState());
  private searchTermSubject = new Subject<string>();
  private filterChangeSubject = new Subject<Partial<SearchFilters>>();
  private sortChangeSubject = new Subject<SortOption>();
  private paginationSubject = new Subject<PaginationOptions>();
  private destroySubject = new Subject<void>();

  // Public observables
  public state$ = this.searchStateSubject.asObservable();
  public results$ = this.state$.pipe(map(state => state.results));
  public isLoading$ = this.state$.pipe(map(state => state.isLoading));
  public error$ = this.state$.pipe(map(state => state.error));
  public query$ = this.state$.pipe(map(state => state.query));

  // Search configuration
  private readonly config = {
    debounceTime: 300,
    minSearchLength: 2,
    maxResults: 50,
    cacheSize: 100,
    cacheTTL: 300000 // 5 minutes
  };

  private searchCache = new Map<string, { result: SearchResult; timestamp: number }>();

  constructor(
    private http: HttpClient,
    private searchAnalytics: SearchAnalyticsService,
    private searchHistory: SearchHistoryService
  ) {
    this.initializeSearchStreams();
  }

  // Search with term
  search(term: string, immediate = false): void {
    if (immediate) {
      this.performSearch(term).subscribe();
    } else {
      this.searchTermSubject.next(term);
    }
  }

  // Update filters
  updateFilters(filters: Partial<SearchFilters>): void {
    this.filterChangeSubject.next(filters);
  }

  // Update sort
  updateSort(sort: SortOption): void {
    this.sortChangeSubject.next(sort);
  }

  // Update pagination
  updatePagination(pagination: PaginationOptions): void {
    this.paginationSubject.next(pagination);
  }

  // Clear search
  clearSearch(): void {
    this.updateState({
      query: this.getDefaultQuery(),
      results: null,
      isLoading: false,
      error: null,
      lastSearchTime: null
    });
  }

  // Get search suggestions
  getSuggestions(term: string): Observable<string[]> {
    if (term.length < this.config.minSearchLength) {
      return of([]);
    }

    return this.http.get<string[]>(`/api/search/suggestions`, {
      params: { q: term, limit: '10' }
    }).pipe(
      catchError(() => of([])),
      shareReplay(1)
    );
  }

  // Advanced search with complex query
  advancedSearch(searchQuery: Partial<SearchQuery>): Observable<SearchResult> {
    const fullQuery = this.buildCompleteQuery(searchQuery);
    return this.executeSearch(fullQuery);
  }

  // Load more results (pagination)
  loadMore(): Observable<SearchResult> {
    const currentState = this.searchStateSubject.value;
    if (!currentState.results?.hasMore || currentState.isLoading) {
      return EMPTY;
    }

    const nextPage = {
      ...currentState.query.pagination,
      page: currentState.query.pagination.page + 1
    };

    const query = { ...currentState.query, pagination: nextPage };
    
    return this.executeSearch(query).pipe(
      tap(result => {
        const currentResults = currentState.results;
        if (currentResults) {
          const combinedResult: SearchResult = {
            ...result,
            items: [...currentResults.items, ...result.items]
          };
          
          this.updateState({
            ...currentState,
            results: combinedResult,
            query,
            isLoading: false
          });
        }
      })
    );
  }

  private initializeSearchStreams(): void {
    // Debounced search stream
    const debouncedSearch$ = this.searchTermSubject.pipe(
      filter(term => term.length >= this.config.minSearchLength || term.length === 0),
      debounceTime(this.config.debounceTime),
      distinctUntilChanged(),
      switchMap(term => this.performSearch(term))
    );

    // Filter change stream
    const filterChange$ = this.filterChangeSubject.pipe(
      withLatestFrom(this.state$),
      map(([filters, state]) => ({
        ...state.query,
        filters: { ...state.query.filters, ...filters },
        pagination: { ...state.query.pagination, page: 1 } // Reset pagination
      })),
      switchMap(query => this.executeSearch(query))
    );

    // Sort change stream
    const sortChange$ = this.sortChangeSubject.pipe(
      withLatestFrom(this.state$),
      map(([sort, state]) => ({
        ...state.query,
        sort,
        pagination: { ...state.query.pagination, page: 1 } // Reset pagination
      })),
      switchMap(query => this.executeSearch(query))
    );

    // Pagination change stream
    const paginationChange$ = this.paginationSubject.pipe(
      withLatestFrom(this.state$),
      map(([pagination, state]) => ({
        ...state.query,
        pagination
      })),
      switchMap(query => this.executeSearch(query))
    );

    // Combine all search triggers
    merge(
      debouncedSearch$,
      filterChange$,
      sortChange$,
      paginationChange$
    ).pipe(
      takeUntil(this.destroySubject)
    ).subscribe();
  }

  private performSearch(term: string): Observable<SearchResult> {
    const currentState = this.searchStateSubject.value;
    const query: SearchQuery = {
      ...currentState.query,
      term,
      pagination: { ...currentState.query.pagination, page: 1 }
    };

    return this.executeSearch(query);
  }

  private executeSearch(query: SearchQuery): Observable<SearchResult> {
    // Check cache first
    const cacheKey = this.getCacheKey(query);
    const cached = this.getCachedResult(cacheKey);
    if (cached) {
      this.updateState({
        query,
        results: cached,
        isLoading: false,
        error: null,
        lastSearchTime: new Date()
      });
      return of(cached);
    }

    // Set loading state
    this.updateState({
      query,
      results: this.searchStateSubject.value.results,
      isLoading: true,
      error: null,
      lastSearchTime: new Date()
    });

    const startTime = Date.now();

    return this.http.post<SearchResult>('/api/search', query).pipe(
      tap(result => {
        const searchTime = Date.now() - startTime;
        const enhancedResult = { ...result, searchTime };

        // Cache result
        this.cacheResult(cacheKey, enhancedResult);

        // Update state
        this.updateState({
          query,
          results: enhancedResult,
          isLoading: false,
          error: null,
          lastSearchTime: new Date()
        });

        // Track analytics
        this.searchAnalytics.trackSearch(query, enhancedResult);

        // Save to history
        if (query.term) {
          this.searchHistory.addToHistory(query.term);
        }
      }),
      catchError(error => {
        this.updateState({
          query,
          results: null,
          isLoading: false,
          error: error.message || 'Search failed',
          lastSearchTime: new Date()
        });
        return throwError(() => error);
      })
    );
  }

  private buildCompleteQuery(partial: Partial<SearchQuery>): SearchQuery {
    const currentState = this.searchStateSubject.value;
    return {
      term: partial.term || currentState.query.term,
      filters: { ...currentState.query.filters, ...partial.filters },
      sort: partial.sort || currentState.query.sort,
      pagination: { ...currentState.query.pagination, ...partial.pagination },
      facets: partial.facets || currentState.query.facets
    };
  }

  private getCacheKey(query: SearchQuery): string {
    return JSON.stringify({
      term: query.term,
      filters: query.filters,
      sort: query.sort,
      pagination: query.pagination,
      facets: query.facets
    });
  }

  private getCachedResult(key: string): SearchResult | null {
    const cached = this.searchCache.get(key);
    if (cached && Date.now() - cached.timestamp < this.config.cacheTTL) {
      return cached.result;
    }
    if (cached) {
      this.searchCache.delete(key);
    }
    return null;
  }

  private cacheResult(key: string, result: SearchResult): void {
    if (this.searchCache.size >= this.config.cacheSize) {
      // Remove oldest entry
      const firstKey = this.searchCache.keys().next().value;
      this.searchCache.delete(firstKey);
    }
    this.searchCache.set(key, { result, timestamp: Date.now() });
  }

  private getInitialState(): SearchState {
    return {
      query: this.getDefaultQuery(),
      results: null,
      isLoading: false,
      error: null,
      lastSearchTime: null
    };
  }

  private getDefaultQuery(): SearchQuery {
    return {
      term: '',
      filters: {},
      sort: { field: 'relevance', direction: 'desc' },
      pagination: { page: 1, size: 20 },
      facets: []
    };
  }

  private updateState(newState: Partial<SearchState>): void {
    const currentState = this.searchStateSubject.value;
    this.searchStateSubject.next({ ...currentState, ...newState });
  }

  ngOnDestroy(): void {
    this.destroySubject.next();
    this.destroySubject.complete();
  }
}

// Additional interfaces
interface DateRange {
  start: Date;
  end: Date;
}

interface PriceRange {
  min: number;
  max: number;
}

interface LocationFilter {
  latitude: number;
  longitude: number;
  radius: number;
  unit: 'km' | 'miles';
}
```

## Advanced Autocomplete Service

### Intelligent Autocomplete with Ranking

```typescript
// autocomplete.service.ts
@Injectable({
  providedIn: 'root'
})
export class AutocompleteService {
  private suggestionCache = new Map<string, AutocompleteResult>();
  private recentSearches: string[] = [];
  private popularSearches: string[] = [];

  constructor(
    private http: HttpClient,
    private searchHistory: SearchHistoryService,
    private searchAnalytics: SearchAnalyticsService
  ) {
    this.loadPopularSearches();
  }

  // Get autocomplete suggestions
  getAutocompleteSuggestions(
    term: string, 
    options: AutocompleteOptions = {}
  ): Observable<AutocompleteResult> {
    if (term.length < (options.minLength || 1)) {
      return of(this.getEmptyResult());
    }

    // Check cache first
    const cacheKey = this.getCacheKey(term, options);
    const cached = this.suggestionCache.get(cacheKey);
    if (cached) {
      return of(cached);
    }

    return this.fetchSuggestions(term, options).pipe(
      map(result => this.enhanceWithRanking(result, term)),
      tap(result => this.suggestionCache.set(cacheKey, result)),
      catchError(() => of(this.getEmptyResult()))
    );
  }

  // Get contextual suggestions
  getContextualSuggestions(
    term: string, 
    context: SearchContext
  ): Observable<AutocompleteResult> {
    const options: AutocompleteOptions = {
      includeHistory: true,
      includePopular: true,
      maxSuggestions: 10,
      categories: context.categories,
      userPreferences: context.userPreferences
    };

    return this.getAutocompleteSuggestions(term, options);
  }

  // Get smart suggestions (ML-enhanced)
  getSmartSuggestions(
    term: string,
    userContext: UserContext
  ): Observable<SmartSuggestionResult> {
    return this.http.post<SmartSuggestionResult>('/api/search/smart-suggestions', {
      term,
      userContext,
      timestamp: new Date().toISOString()
    }).pipe(
      catchError(() => of({
        suggestions: [],
        entities: [],
        intent: 'unknown',
        confidence: 0
      }))
    );
  }

  // Track suggestion selection
  trackSuggestionSelection(suggestion: string, position: number): void {
    this.searchAnalytics.trackSuggestionClick(suggestion, position);
    this.addToRecentSearches(suggestion);
  }

  // Get instant search results
  getInstantResults(term: string): Observable<InstantSearchResult> {
    if (term.length < 3) {
      return of({ quickResults: [], hasMore: false });
    }

    return this.http.get<InstantSearchResult>('/api/search/instant', {
      params: { q: term, limit: '5' }
    }).pipe(
      debounceTime(150),
      distinctUntilChanged(),
      catchError(() => of({ quickResults: [], hasMore: false }))
    );
  }

  private fetchSuggestions(
    term: string, 
    options: AutocompleteOptions
  ): Observable<AutocompleteResult> {
    const params = new HttpParams()
      .set('q', term)
      .set('max', (options.maxSuggestions || 10).toString())
      .set('includeHistory', (options.includeHistory || false).toString())
      .set('includePopular', (options.includePopular || false).toString());

    return this.http.get<AutocompleteApiResponse>('/api/search/autocomplete', { params }).pipe(
      map(response => this.mapApiResponse(response, options))
    );
  }

  private enhanceWithRanking(result: AutocompleteResult, term: string): AutocompleteResult {
    const rankedSuggestions = result.suggestions.map(suggestion => ({
      ...suggestion,
      score: this.calculateRelevanceScore(suggestion, term)
    })).sort((a, b) => b.score - a.score);

    return {
      ...result,
      suggestions: rankedSuggestions
    };
  }

  private calculateRelevanceScore(suggestion: AutocompleteSuggestion, term: string): number {
    let score = 0;
    const lowerTerm = term.toLowerCase();
    const lowerSuggestion = suggestion.text.toLowerCase();

    // Exact match gets highest score
    if (lowerSuggestion === lowerTerm) {
      score += 100;
    }

    // Starts with term gets high score
    if (lowerSuggestion.startsWith(lowerTerm)) {
      score += 80;
    }

    // Contains term gets medium score
    if (lowerSuggestion.includes(lowerTerm)) {
      score += 50;
    }

    // Boost based on type
    const typeBoosts: { [key in SuggestionType]: number } = {
      'query': 10,
      'product': 15,
      'category': 8,
      'brand': 12,
      'history': 5,
      'popular': 7
    };
    score += typeBoosts[suggestion.type] || 0;

    // Boost recent searches
    if (this.recentSearches.includes(suggestion.text)) {
      score += 20;
    }

    // Boost popular searches
    if (this.popularSearches.includes(suggestion.text)) {
      score += 15;
    }

    // Boost based on frequency/popularity
    score += Math.min(suggestion.frequency || 0, 20);

    return score;
  }

  private mapApiResponse(
    response: AutocompleteApiResponse, 
    options: AutocompleteOptions
  ): AutocompleteResult {
    let suggestions: AutocompleteSuggestion[] = response.suggestions.map(s => ({
      text: s.text,
      type: s.type,
      category: s.category,
      frequency: s.frequency,
      score: 0,
      metadata: s.metadata
    }));

    // Add history suggestions if requested
    if (options.includeHistory) {
      const historySuggestions = this.getHistorySuggestions(options.maxSuggestions || 10);
      suggestions = [...suggestions, ...historySuggestions];
    }

    // Add popular suggestions if requested
    if (options.includePopular) {
      const popularSuggestions = this.getPopularSuggestions(options.maxSuggestions || 10);
      suggestions = [...suggestions, ...popularSuggestions];
    }

    // Remove duplicates
    const uniqueSuggestions = this.removeDuplicates(suggestions);

    return {
      suggestions: uniqueSuggestions.slice(0, options.maxSuggestions || 10),
      categories: response.categories || [],
      hasMore: response.hasMore || false
    };
  }

  private getHistorySuggestions(maxCount: number): AutocompleteSuggestion[] {
    return this.recentSearches.slice(0, maxCount).map(text => ({
      text,
      type: 'history',
      category: 'recent',
      frequency: 0,
      score: 0,
      metadata: { isHistory: true }
    }));
  }

  private getPopularSuggestions(maxCount: number): AutocompleteSuggestion[] {
    return this.popularSearches.slice(0, maxCount).map(text => ({
      text,
      type: 'popular',
      category: 'trending',
      frequency: 0,
      score: 0,
      metadata: { isPopular: true }
    }));
  }

  private removeDuplicates(suggestions: AutocompleteSuggestion[]): AutocompleteSuggestion[] {
    const seen = new Set<string>();
    return suggestions.filter(suggestion => {
      if (seen.has(suggestion.text.toLowerCase())) {
        return false;
      }
      seen.add(suggestion.text.toLowerCase());
      return true;
    });
  }

  private addToRecentSearches(term: string): void {
    this.recentSearches = [term, ...this.recentSearches.filter(t => t !== term)].slice(0, 10);
    localStorage.setItem('recentSearches', JSON.stringify(this.recentSearches));
  }

  private loadPopularSearches(): void {
    this.http.get<string[]>('/api/search/popular').subscribe({
      next: (searches) => this.popularSearches = searches,
      error: () => this.popularSearches = []
    });
  }

  private getCacheKey(term: string, options: AutocompleteOptions): string {
    return `${term}:${JSON.stringify(options)}`;
  }

  private getEmptyResult(): AutocompleteResult {
    return {
      suggestions: [],
      categories: [],
      hasMore: false
    };
  }
}

// Interfaces
interface AutocompleteOptions {
  minLength?: number;
  maxSuggestions?: number;
  includeHistory?: boolean;
  includePopular?: boolean;
  categories?: string[];
  userPreferences?: any;
}

interface AutocompleteResult {
  suggestions: AutocompleteSuggestion[];
  categories: string[];
  hasMore: boolean;
}

interface AutocompleteSuggestion {
  text: string;
  type: SuggestionType;
  category?: string;
  frequency?: number;
  score: number;
  metadata?: any;
}

type SuggestionType = 'query' | 'product' | 'category' | 'brand' | 'history' | 'popular';

interface AutocompleteApiResponse {
  suggestions: {
    text: string;
    type: SuggestionType;
    category?: string;
    frequency?: number;
    metadata?: any;
  }[];
  categories?: string[];
  hasMore?: boolean;
}

interface SearchContext {
  categories: string[];
  userPreferences: any;
  location?: LocationFilter;
  previousSearches?: string[];
}

interface UserContext {
  userId?: string;
  sessionId: string;
  preferences: any;
  searchHistory: string[];
  location?: LocationFilter;
}

interface SmartSuggestionResult {
  suggestions: AutocompleteSuggestion[];
  entities: DetectedEntity[];
  intent: SearchIntent;
  confidence: number;
}

interface DetectedEntity {
  type: 'product' | 'brand' | 'category' | 'location' | 'person';
  value: string;
  confidence: number;
}

type SearchIntent = 'product_search' | 'information_seeking' | 'navigation' | 'comparison' | 'unknown';

interface InstantSearchResult {
  quickResults: QuickResult[];
  hasMore: boolean;
}

interface QuickResult {
  id: string;
  title: string;
  type: 'product' | 'article' | 'page';
  url: string;
  image?: string;
  price?: number;
}
```

## Search State Management

### Advanced Search State with URL Synchronization

```typescript
// search-state.service.ts
@Injectable({
  providedIn: 'root'
})
export class SearchStateService {
  private stateSubject = new BehaviorSubject<CompleteSearchState>(this.getInitialState());
  private urlSyncEnabled = true;

  public state$ = this.stateSubject.asObservable();
  public searchParams$ = this.state$.pipe(map(state => state.searchParams));
  public ui$ = this.state$.pipe(map(state => state.ui));
  public metadata$ = this.state$.pipe(map(state => state.metadata));

  constructor(
    private router: Router,
    private route: ActivatedRoute,
    private location: Location
  ) {
    this.initializeFromUrl();
    this.setupUrlSync();
  }

  // Update search parameters
  updateSearchParams(updates: Partial<SearchParameters>): void {
    const currentState = this.stateSubject.value;
    const newParams = { ...currentState.searchParams, ...updates };
    
    this.updateState({
      searchParams: newParams,
      metadata: {
        ...currentState.metadata,
        lastUpdated: new Date(),
        hasChanged: true
      }
    });

    if (this.urlSyncEnabled) {
      this.syncToUrl(newParams);
    }
  }

  // Update UI state
  updateUIState(updates: Partial<SearchUIState>): void {
    const currentState = this.stateSubject.value;
    this.updateState({
      ui: { ...currentState.ui, ...updates }
    });
  }

  // Reset search state
  resetState(): void {
    const initialState = this.getInitialState();
    this.stateSubject.next(initialState);
    
    if (this.urlSyncEnabled) {
      this.clearUrl();
    }
  }

  // Save search state
  saveState(name: string): Observable<SavedSearch> {
    const currentState = this.stateSubject.value;
    const savedSearch: SavedSearch = {
      id: this.generateId(),
      name,
      searchParams: currentState.searchParams,
      createdAt: new Date(),
      lastUsed: new Date()
    };

    return this.http.post<SavedSearch>('/api/search/saved', savedSearch).pipe(
      tap(() => {
        this.updateState({
          metadata: {
            ...currentState.metadata,
            savedSearches: [...currentState.metadata.savedSearches, savedSearch]
          }
        });
      })
    );
  }

  // Load saved search
  loadSavedSearch(savedSearchId: string): Observable<void> {
    const currentState = this.stateSubject.value;
    const savedSearch = currentState.metadata.savedSearches.find(s => s.id === savedSearchId);
    
    if (savedSearch) {
      this.updateSearchParams(savedSearch.searchParams);
      return of(void 0);
    }

    return this.http.get<SavedSearch>(`/api/search/saved/${savedSearchId}`).pipe(
      tap(savedSearch => {
        this.updateSearchParams(savedSearch.searchParams);
      }),
      map(() => void 0)
    );
  }

  // Get state as URL parameters
  getUrlParams(): { [key: string]: any } {
    const state = this.stateSubject.value;
    return this.serializeSearchParams(state.searchParams);
  }

  // Load state from URL parameters
  loadFromUrlParams(params: { [key: string]: any }): void {
    const searchParams = this.deserializeSearchParams(params);
    this.updateSearchParams(searchParams);
  }

  // Enable/disable URL synchronization
  setUrlSyncEnabled(enabled: boolean): void {
    this.urlSyncEnabled = enabled;
  }

  // Get search permalink
  getSearchPermalink(): string {
    const params = this.getUrlParams();
    const url = this.router.createUrlTree([], { 
      queryParams: params,
      relativeTo: this.route 
    });
    return this.location.prepareExternalUrl(url.toString());
  }

  private initializeFromUrl(): void {
    this.route.queryParams.pipe(take(1)).subscribe(params => {
      if (Object.keys(params).length > 0) {
        this.loadFromUrlParams(params);
      }
    });
  }

  private setupUrlSync(): void {
    // Sync state changes to URL with debouncing
    this.searchParams$.pipe(
      debounceTime(500),
      distinctUntilChanged((a, b) => JSON.stringify(a) === JSON.stringify(b)),
      filter(() => this.urlSyncEnabled)
    ).subscribe(params => {
      this.syncToUrl(params);
    });
  }

  private syncToUrl(params: SearchParameters): void {
    const queryParams = this.serializeSearchParams(params);
    
    this.router.navigate([], {
      relativeTo: this.route,
      queryParams,
      queryParamsHandling: 'replace',
      replaceUrl: true
    });
  }

  private clearUrl(): void {
    this.router.navigate([], {
      relativeTo: this.route,
      queryParams: {},
      replaceUrl: true
    });
  }

  private serializeSearchParams(params: SearchParameters): { [key: string]: any } {
    const serialized: { [key: string]: any } = {};

    if (params.query) serialized.q = params.query;
    if (params.category?.length) serialized.category = params.category;
    if (params.priceRange) {
      serialized.minPrice = params.priceRange.min;
      serialized.maxPrice = params.priceRange.max;
    }
    if (params.dateRange) {
      serialized.startDate = params.dateRange.start.toISOString();
      serialized.endDate = params.dateRange.end.toISOString();
    }
    if (params.tags?.length) serialized.tags = params.tags.join(',');
    if (params.sort) {
      serialized.sort = params.sort.field;
      serialized.order = params.sort.direction;
    }
    if (params.page > 1) serialized.page = params.page;
    if (params.size !== 20) serialized.size = params.size;

    return serialized;
  }

  private deserializeSearchParams(params: { [key: string]: any }): SearchParameters {
    const searchParams: SearchParameters = this.getDefaultSearchParams();

    if (params.q) searchParams.query = params.q;
    if (params.category) {
      searchParams.category = Array.isArray(params.category) 
        ? params.category 
        : [params.category];
    }
    if (params.minPrice && params.maxPrice) {
      searchParams.priceRange = {
        min: Number(params.minPrice),
        max: Number(params.maxPrice)
      };
    }
    if (params.startDate && params.endDate) {
      searchParams.dateRange = {
        start: new Date(params.startDate),
        end: new Date(params.endDate)
      };
    }
    if (params.tags) {
      searchParams.tags = typeof params.tags === 'string' 
        ? params.tags.split(',') 
        : params.tags;
    }
    if (params.sort) {
      searchParams.sort = {
        field: params.sort,
        direction: params.order || 'desc'
      };
    }
    if (params.page) searchParams.page = Number(params.page);
    if (params.size) searchParams.size = Number(params.size);

    return searchParams;
  }

  private updateState(updates: Partial<CompleteSearchState>): void {
    const currentState = this.stateSubject.value;
    this.stateSubject.next({ ...currentState, ...updates });
  }

  private getInitialState(): CompleteSearchState {
    return {
      searchParams: this.getDefaultSearchParams(),
      ui: {
        viewMode: 'grid',
        showFilters: false,
        showSort: false,
        isAdvancedMode: false,
        selectedItems: [],
        focusedItem: null
      },
      metadata: {
        lastUpdated: new Date(),
        hasChanged: false,
        savedSearches: [],
        searchHistory: [],
        totalSearches: 0
      }
    };
  }

  private getDefaultSearchParams(): SearchParameters {
    return {
      query: '',
      category: [],
      priceRange: null,
      dateRange: null,
      tags: [],
      sort: { field: 'relevance', direction: 'desc' },
      page: 1,
      size: 20
    };
  }

  private generateId(): string {
    return `search-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  }
}

// Interfaces
interface CompleteSearchState {
  searchParams: SearchParameters;
  ui: SearchUIState;
  metadata: SearchMetadata;
}

interface SearchParameters {
  query: string;
  category: string[];
  priceRange: PriceRange | null;
  dateRange: DateRange | null;
  tags: string[];
  sort: SortOption;
  page: number;
  size: number;
}

interface SearchUIState {
  viewMode: 'grid' | 'list' | 'compact';
  showFilters: boolean;
  showSort: boolean;
  isAdvancedMode: boolean;
  selectedItems: string[];
  focusedItem: string | null;
}

interface SearchMetadata {
  lastUpdated: Date;
  hasChanged: boolean;
  savedSearches: SavedSearch[];
  searchHistory: string[];
  totalSearches: number;
}

interface SavedSearch {
  id: string;
  name: string;
  searchParams: SearchParameters;
  createdAt: Date;
  lastUsed: Date;
}
```

## Faceted Search & Filtering

### Advanced Faceted Search Implementation

```typescript
// faceted-search.service.ts
@Injectable({
  providedIn: 'root'
})
export class FacetedSearchService {
  private facetsSubject = new BehaviorSubject<FacetCollection>({ facets: [], activeFilters: {} });
  private filtersSubject = new BehaviorSubject<ActiveFilters>({});

  public facets$ = this.facetsSubject.asObservable();
  public activeFilters$ = this.filtersSubject.asObservable();
  public filterCount$ = this.activeFilters$.pipe(
    map(filters => Object.values(filters).flat().length)
  );

  constructor(
    private http: HttpClient,
    private searchService: SearchService
  ) {
    this.setupFilterSync();
  }

  // Load facets for search results
  loadFacets(searchQuery: SearchQuery): Observable<SearchFacet[]> {
    return this.http.post<SearchFacet[]>('/api/search/facets', {
      query: searchQuery.term,
      filters: searchQuery.filters,
      excludeFilters: true // Get all possible facets
    }).pipe(
      tap(facets => {
        const currentState = this.facetsSubject.value;
        this.facetsSubject.next({
          ...currentState,
          facets: this.enhanceFacets(facets)
        });
      }),
      shareReplay(1)
    );
  }

  // Apply filter
  applyFilter(facetName: string, value: string | FacetRange): void {
    const currentFilters = this.filtersSubject.value;
    const existingValues = currentFilters[facetName] || [];

    let newValues: (string | FacetRange)[];
    
    if (typeof value === 'string') {
      newValues = existingValues.includes(value)
        ? existingValues.filter(v => v !== value)
        : [...existingValues, value];
    } else {
      // Range filter - replace existing range
      newValues = [value];
    }

    const updatedFilters = {
      ...currentFilters,
      [facetName]: newValues.length > 0 ? newValues : undefined
    };

    // Remove empty filter arrays
    Object.keys(updatedFilters).forEach(key => {
      if (!updatedFilters[key] || updatedFilters[key].length === 0) {
        delete updatedFilters[key];
      }
    });

    this.filtersSubject.next(updatedFilters);
    this.updateFacetState(updatedFilters);
  }

  // Remove filter
  removeFilter(facetName: string, value?: string | FacetRange): void {
    const currentFilters = this.filtersSubject.value;
    
    if (!value) {
      // Remove entire facet
      const { [facetName]: removed, ...remaining } = currentFilters;
      this.filtersSubject.next(remaining);
    } else {
      // Remove specific value
      const existingValues = currentFilters[facetName] || [];
      const newValues = existingValues.filter(v => 
        typeof v === 'string' && typeof value === 'string' 
          ? v !== value 
          : JSON.stringify(v) !== JSON.stringify(value)
      );
      
      this.applyFilter(facetName, newValues[0]); // This will handle the logic
    }
  }

  // Clear all filters
  clearAllFilters(): void {
    this.filtersSubject.next({});
    this.updateFacetState({});
  }

  // Clear filters for specific facet
  clearFacetFilters(facetName: string): void {
    const currentFilters = this.filtersSubject.value;
    const { [facetName]: removed, ...remaining } = currentFilters;
    this.filtersSubject.next(remaining);
    this.updateFacetState(remaining);
  }

  // Get filter suggestions for autocomplete facets
  getFacetSuggestions(facetName: string, term: string): Observable<FacetSuggestion[]> {
    if (!term || term.length < 2) {
      return of([]);
    }

    return this.http.get<FacetSuggestion[]>('/api/search/facet-suggestions', {
      params: {
        facet: facetName,
        term,
        limit: '10'
      }
    }).pipe(
      catchError(() => of([]))
    );
  }

  // Build filter summary for display
  getFilterSummary(): Observable<FilterSummary[]> {
    return combineLatest([this.facets$, this.activeFilters$]).pipe(
      map(([facetCollection, activeFilters]) => {
        const summary: FilterSummary[] = [];
        
        Object.entries(activeFilters).forEach(([facetName, values]) => {
          const facet = facetCollection.facets.find(f => f.name === facetName);
          if (facet && values.length > 0) {
            const filterSummary: FilterSummary = {
              facetName,
              facetLabel: facet.label || facetName,
              values: values.map(value => ({
                value,
                label: this.getValueLabel(facet, value),
                canRemove: true
              }))
            };
            summary.push(filterSummary);
          }
        });
        
        return summary;
      })
    );
  }

  // Advanced facet search with dynamic loading
  searchFacetValues(
    facetName: string, 
    searchTerm: string, 
    limit = 20
  ): Observable<FacetValue[]> {
    const currentFilters = this.filtersSubject.value;
    
    return this.http.post<FacetValue[]>('/api/search/facet-values', {
      facetName,
      searchTerm,
      excludeFilters: currentFilters,
      limit
    }).pipe(
      map(values => values.map(value => ({
        ...value,
        selected: this.isValueSelected(facetName, value.value)
      }))),
      catchError(() => of([]))
    );
  }

  private enhanceFacets(facets: SearchFacet[]): SearchFacet[] {
    const activeFilters = this.filtersSubject.value;
    
    return facets.map(facet => ({
      ...facet,
      values: facet.values.map(value => ({
        ...value,
        selected: this.isValueSelected(facet.name, value.value)
      }))
    }));
  }

  private isValueSelected(facetName: string, value: string): boolean {
    const activeFilters = this.filtersSubject.value;
    const facetFilters = activeFilters[facetName] || [];
    return facetFilters.some(filter => 
      typeof filter === 'string' ? filter === value : false
    );
  }

  private updateFacetState(activeFilters: ActiveFilters): void {
    const currentState = this.facetsSubject.value;
    this.facetsSubject.next({
      ...currentState,
      activeFilters,
      facets: this.enhanceFacets(currentState.facets)
    });
  }

  private setupFilterSync(): void {
    // Sync filters with search service
    this.activeFilters$.subscribe(filters => {
      const searchFilters = this.convertToSearchFilters(filters);
      this.searchService.updateFilters(searchFilters);
    });
  }

  private convertToSearchFilters(activeFilters: ActiveFilters): SearchFilters {
    const searchFilters: SearchFilters = {};
    
    Object.entries(activeFilters).forEach(([facetName, values]) => {
      if (values.length > 0) {
        if (facetName === 'price') {
          const range = values[0] as FacetRange;
          searchFilters.priceRange = { min: range.min, max: range.max };
        } else if (facetName === 'date') {
          const range = values[0] as FacetRange;
          searchFilters.dateRange = {
            start: new Date(range.min),
            end: new Date(range.max)
          };
        } else {
          searchFilters[facetName] = values as string[];
        }
      }
    });
    
    return searchFilters;
  }

  private getValueLabel(facet: SearchFacet, value: string | FacetRange): string {
    if (typeof value === 'string') {
      const facetValue = facet.values.find(v => v.value === value);
      return facetValue?.label || value;
    } else {
      return `${value.min} - ${value.max}`;
    }
  }
}

// Interfaces
interface FacetCollection {
  facets: SearchFacet[];
  activeFilters: ActiveFilters;
}

interface ActiveFilters {
  [facetName: string]: (string | FacetRange)[];
}

interface FacetRange {
  min: number;
  max: number;
}

interface FacetSuggestion {
  value: string;
  label: string;
  count: number;
}

interface FilterSummary {
  facetName: string;
  facetLabel: string;
  values: FilterValue[];
}

interface FilterValue {
  value: string | FacetRange;
  label: string;
  canRemove: boolean;
}

// Enhanced SearchFacet interface
interface SearchFacet {
  name: string;
  label?: string;
  type: 'checkbox' | 'range' | 'select' | 'autocomplete' | 'color' | 'rating';
  values: FacetValue[];
  multiSelect?: boolean;
  searchable?: boolean;
  collapsible?: boolean;
  collapsed?: boolean;
  order?: number;
}

interface FacetValue {
  value: string;
  label?: string;
  count: number;
  selected: boolean;
  color?: string; // For color facets
  image?: string; // For image-based facets
}
```

## Summary

This lesson covered comprehensive search and autocomplete patterns:

### Key Takeaways:
1. **Search Architecture**: Complete search infrastructure with caching, debouncing, and state management
2. **Advanced Autocomplete**: Intelligent suggestions with ranking, context awareness, and ML enhancement
3. **State Management**: URL synchronization, saved searches, and comprehensive state handling
4. **Faceted Search**: Dynamic filtering with multiple facet types and real-time updates
5. **Performance**: Caching strategies, debouncing, and optimization techniques
6. **User Experience**: Instant results, smart suggestions, and intuitive filtering
7. **Analytics Integration**: Search tracking and user behavior analysis
8. **Accessibility**: Keyboard navigation and screen reader support

### Next Steps:
- Implement search systems in your applications
- Add analytics and performance monitoring
- Consider A/B testing for search experiences
- Optimize for mobile and touch interfaces

In the next lesson, we'll explore **Drag & Drop Patterns** for building interactive interfaces with RxJS.
