# Mathematical Operators

## Overview

Mathematical operators provide powerful tools for performing calculations, aggregations, and statistical operations on Observable streams. These operators allow you to compute sums, averages, find minimum and maximum values, count emissions, and perform complex mathematical transformations on reactive data streams, making them essential for analytics, monitoring, and data processing applications.

## Learning Objectives

After completing this lesson, you will be able to:
- Perform mathematical calculations on Observable streams
- Implement aggregation and statistical operations
- Build real-time analytics and monitoring systems
- Apply mathematical operators in Angular applications
- Optimize performance for large-scale data processing

## Core Mathematical Operators

### 1. count() - Count Emissions

The `count()` operator counts the number of emissions from the source Observable and emits that count when the source completes.

```typescript
import { of, range, interval } from 'rxjs';
import { count, take, filter } from 'rxjs/operators';

// Basic counting
const numbers$ = of(1, 2, 3, 4, 5);
const totalCount$ = numbers$.pipe(count());
totalCount$.subscribe(count => console.log('Total count:', count)); // 5

// Conditional counting
const evenCount$ = numbers$.pipe(
  count(n => n % 2 === 0)
);
evenCount$.subscribe(count => console.log('Even count:', count)); // 2

// Angular analytics service
@Injectable()
export class AnalyticsService {
  private userActions$ = new Subject<UserAction>();

  // Count user interactions by type
  getActionCounts(): Observable<ActionCounts> {
    return this.userActions$.pipe(
      bufferTime(60000), // Collect actions for 1 minute
      switchMap(actions => 
        forkJoin({
          total: of(actions).pipe(
            mergeMap(arr => from(arr)),
            count()
          ),
          clicks: of(actions).pipe(
            mergeMap(arr => from(arr)),
            count(action => action.type === 'click')
          ),
          views: of(actions).pipe(
            mergeMap(arr => from(arr)),
            count(action => action.type === 'view')
          ),
          purchases: of(actions).pipe(
            mergeMap(arr => from(arr)),
            count(action => action.type === 'purchase')
          )
        })
      )
    );
  }

  trackAction(action: UserAction) {
    this.userActions$.next(action);
  }
}

// Performance monitoring
@Component({})
export class PerformanceMonitorComponent implements OnInit {
  private apiCalls$ = new Subject<ApiCall>();

  apiMetrics$ = this.apiCalls$.pipe(
    bufferTime(30000), // 30-second windows
    switchMap(calls => 
      forkJoin({
        totalCalls: of(calls).pipe(
          mergeMap(arr => from(arr)),
          count()
        ),
        successfulCalls: of(calls).pipe(
          mergeMap(arr => from(arr)),
          count(call => call.status >= 200 && call.status < 300)
        ),
        errorCalls: of(calls).pipe(
          mergeMap(arr => from(arr)),
          count(call => call.status >= 400)
        ),
        slowCalls: of(calls).pipe(
          mergeMap(arr => from(arr)),
          count(call => call.duration > 5000)
        )
      })
    ),
    map(counts => ({
      ...counts,
      successRate: counts.totalCalls > 0 ? counts.successfulCalls / counts.totalCalls : 0,
      errorRate: counts.totalCalls > 0 ? counts.errorCalls / counts.totalCalls : 0
    }))
  );

  ngOnInit() {
    // Monitor HTTP interceptor calls
    this.httpInterceptorService.apiCalls$.subscribe(call => {
      this.apiCalls$.next(call);
    });
  }
}
```

**Marble Diagram:**
```
Input:  -1-2-3-4-5|
count()
Output: ----------5|

Input:  -1-2-3-4-5|
count(x => x % 2 === 0)
Output: ----------2|
```

### 2. max() - Find Maximum Value

The `max()` operator finds the maximum value emitted by the source Observable.

```typescript
import { of, from } from 'rxjs';
import { max, map } from 'rxjs/operators';

// Basic maximum
const numbers$ = of(3, 7, 2, 9, 1, 8);
const maximum$ = numbers$.pipe(max());
maximum$.subscribe(max => console.log('Maximum:', max)); // 9

// Custom comparison
interface Product {
  id: string;
  name: string;
  price: number;
  rating: number;
}

const products$ = of(
  { id: '1', name: 'Laptop', price: 1200, rating: 4.5 },
  { id: '2', name: 'Phone', price: 800, rating: 4.8 },
  { id: '3', name: 'Tablet', price: 600, rating: 4.2 }
);

const highestPriced$ = products$.pipe(
  max((a, b) => a.price < b.price ? -1 : 1)
);

const topRated$ = products$.pipe(
  max((a, b) => a.rating < b.rating ? -1 : 1)
);

// Angular dashboard metrics
@Injectable()
export class MetricsService {
  constructor(private http: HttpClient) {}

  // Find peak performance metrics
  getPeakMetrics(): Observable<PeakMetrics> {
    return this.http.get<PerformanceData[]>('/api/metrics').pipe(
      mergeMap(data => 
        forkJoin({
          maxCpu: from(data).pipe(
            max((a, b) => a.cpuUsage < b.cpuUsage ? -1 : 1),
            map(peak => ({ value: peak.cpuUsage, timestamp: peak.timestamp }))
          ),
          maxMemory: from(data).pipe(
            max((a, b) => a.memoryUsage < b.memoryUsage ? -1 : 1),
            map(peak => ({ value: peak.memoryUsage, timestamp: peak.timestamp }))
          ),
          maxResponseTime: from(data).pipe(
            max((a, b) => a.responseTime < b.responseTime ? -1 : 1),
            map(peak => ({ value: peak.responseTime, timestamp: peak.timestamp }))
          )
        })
      )
    );
  }

  // Real-time peak tracking
  trackRealtimePeaks(): Observable<RealtimePeak> {
    return interval(5000).pipe(
      switchMap(() => this.getCurrentMetrics()),
      scan((peaks, current) => ({
        cpu: current.cpu > peaks.cpu.value ? 
          { value: current.cpu, timestamp: Date.now() } : peaks.cpu,
        memory: current.memory > peaks.memory.value ? 
          { value: current.memory, timestamp: Date.now() } : peaks.memory,
        network: current.network > peaks.network.value ? 
          { value: current.network, timestamp: Date.now() } : peaks.network
      }), {
        cpu: { value: 0, timestamp: 0 },
        memory: { value: 0, timestamp: 0 },
        network: { value: 0, timestamp: 0 }
      })
    );
  }

  private getCurrentMetrics(): Observable<CurrentMetrics> {
    return this.http.get<CurrentMetrics>('/api/metrics/current');
  }
}
```

### 3. min() - Find Minimum Value

The `min()` operator finds the minimum value emitted by the source Observable.

```typescript
import { of, from } from 'rxjs';
import { min, map } from 'rxjs/operators';

// Basic minimum
const numbers$ = of(3, 7, 2, 9, 1, 8);
const minimum$ = numbers$.pipe(min());
minimum$.subscribe(min => console.log('Minimum:', min)); // 1

// Angular pricing service
@Injectable()
export class PricingService {
  constructor(private http: HttpClient) {}

  findBestDeals(): Observable<BestDeals> {
    return this.http.get<Product[]>('/api/products').pipe(
      mergeMap(products => 
        forkJoin({
          cheapestOverall: from(products).pipe(
            min((a, b) => a.price < b.price ? -1 : 1)
          ),
          cheapestByCategory: this.findCheapestByCategory(products),
          bestValueForMoney: from(products).pipe(
            min((a, b) => (a.price / a.rating) < (b.price / b.rating) ? -1 : 1)
          )
        })
      )
    );
  }

  private findCheapestByCategory(products: Product[]): Observable<CategoryDeals> {
    const categories = [...new Set(products.map(p => p.category))];
    
    return from(categories).pipe(
      mergeMap(category => {
        const categoryProducts = products.filter(p => p.category === category);
        return from(categoryProducts).pipe(
          min((a, b) => a.price < b.price ? -1 : 1),
          map(cheapest => ({ category, product: cheapest }))
        );
      }),
      toArray(),
      map(deals => deals.reduce((acc, deal) => {
        acc[deal.category] = deal.product;
        return acc;
      }, {} as CategoryDeals))
    );
  }

  // Monitor minimum system requirements
  checkSystemRequirements(): Observable<SystemCheck> {
    return this.systemService.getSystemInfo().pipe(
      map(info => ({
        meetsMinimum: this.checkMinimumRequirements(info),
        recommendations: this.getRecommendations(info)
      }))
    );
  }

  private checkMinimumRequirements(info: SystemInfo): boolean {
    const requirements = {
      ram: 4, // GB
      storage: 10, // GB
      cpuCores: 2
    };

    return info.ram >= requirements.ram &&
           info.storage >= requirements.storage &&
           info.cpuCores >= requirements.cpuCores;
  }

  private getRecommendations(info: SystemInfo): string[] {
    const recommendations: string[] = [];
    
    if (info.ram < 8) recommendations.push('Consider upgrading RAM to 8GB');
    if (info.storage < 50) recommendations.push('Free up disk space');
    if (info.cpuCores < 4) recommendations.push('CPU upgrade recommended');
    
    return recommendations;
  }
}
```

### 4. reduce() - Accumulate Values

The `reduce()` operator applies an accumulator function over the source Observable and returns the accumulated result when the source completes.

```typescript
import { of, from } from 'rxjs';
import { reduce, map } from 'rxjs/operators';

// Basic reduction
const numbers$ = of(1, 2, 3, 4, 5);
const sum$ = numbers$.pipe(
  reduce((acc, value) => acc + value, 0)
);
sum$.subscribe(sum => console.log('Sum:', sum)); // 15

// Complex aggregations
interface SalesData {
  date: string;
  amount: number;
  category: string;
  region: string;
}

// Angular reporting service
@Injectable()
export class ReportingService {
  constructor(private http: HttpClient) {}

  generateSalesReport(dateRange: DateRange): Observable<SalesReport> {
    return this.http.get<SalesData[]>(`/api/sales?from=${dateRange.from}&to=${dateRange.to}`).pipe(
      mergeMap(salesData => 
        forkJoin({
          totalRevenue: from(salesData).pipe(
            reduce((total, sale) => total + sale.amount, 0)
          ),
          averageOrderValue: from(salesData).pipe(
            reduce((acc, sale) => ({
              total: acc.total + sale.amount,
              count: acc.count + 1
            }), { total: 0, count: 0 }),
            map(acc => acc.count > 0 ? acc.total / acc.count : 0)
          ),
          categoryBreakdown: from(salesData).pipe(
            reduce((categories, sale) => {
              categories[sale.category] = (categories[sale.category] || 0) + sale.amount;
              return categories;
            }, {} as Record<string, number>)
          ),
          regionBreakdown: from(salesData).pipe(
            reduce((regions, sale) => {
              regions[sale.region] = (regions[sale.region] || 0) + sale.amount;
              return regions;
            }, {} as Record<string, number>)
          ),
          topPerformingCategory: from(salesData).pipe(
            reduce((categories, sale) => {
              categories[sale.category] = (categories[sale.category] || 0) + sale.amount;
              return categories;
            }, {} as Record<string, number>),
            map(categories => 
              Object.entries(categories).reduce((top, [category, amount]) => 
                amount > top.amount ? { category, amount } : top
              , { category: '', amount: 0 })
            )
          )
        })
      )
    );
  }

  // Real-time aggregation
  getRealTimeMetrics(): Observable<RealTimeMetrics> {
    return this.websocketService.getTransactionStream().pipe(
      bufferTime(10000), // 10-second windows
      switchMap(transactions => 
        from(transactions).pipe(
          reduce((metrics, transaction) => ({
            count: metrics.count + 1,
            totalValue: metrics.totalValue + transaction.amount,
            averageValue: 0, // Will be calculated after
            maxTransaction: Math.max(metrics.maxTransaction, transaction.amount),
            minTransaction: Math.min(metrics.minTransaction, transaction.amount),
            successfulTransactions: metrics.successfulTransactions + 
              (transaction.status === 'success' ? 1 : 0)
          }), {
            count: 0,
            totalValue: 0,
            averageValue: 0,
            maxTransaction: 0,
            minTransaction: Infinity,
            successfulTransactions: 0
          }),
          map(metrics => ({
            ...metrics,
            averageValue: metrics.count > 0 ? metrics.totalValue / metrics.count : 0,
            successRate: metrics.count > 0 ? metrics.successfulTransactions / metrics.count : 0,
            minTransaction: metrics.minTransaction === Infinity ? 0 : metrics.minTransaction
          }))
        )
      )
    );
  }
}
```

### 5. scan() - Intermediate Accumulation

The `scan()` operator is similar to reduce but emits the intermediate accumulated values.

```typescript
import { of, interval } from 'rxjs';
import { scan, map, take } from 'rxjs/operators';

// Running total
const numbers$ = of(1, 2, 3, 4, 5);
const runningTotal$ = numbers$.pipe(
  scan((acc, value) => acc + value, 0)
);
runningTotal$.subscribe(total => console.log('Running total:', total));
// Output: 1, 3, 6, 10, 15

// Angular real-time dashboard
@Component({
  template: `
    <div class="dashboard">
      <div class="metrics-grid">
        <div class="metric-card">
          <h3>Total Visits</h3>
          <div class="metric-value">{{ (cumulativeMetrics$ | async)?.totalVisits | number }}</div>
          <div class="metric-trend" [class]="getTrendClass((cumulativeMetrics$ | async)?.visitsTrend)">
            {{ (cumulativeMetrics$ | async)?.visitsTrend }}%
          </div>
        </div>

        <div class="metric-card">
          <h3>Revenue</h3>
          <div class="metric-value">{{ (cumulativeMetrics$ | async)?.totalRevenue | currency }}</div>
          <div class="metric-change">
            +{{ (cumulativeMetrics$ | async)?.revenueChange | currency }} today
          </div>
        </div>

        <div class="metric-card">
          <h3>Conversion Rate</h3>
          <div class="metric-value">{{ (cumulativeMetrics$ | async)?.conversionRate | percent:'1.2-2' }}</div>
        </div>

        <div class="metric-card">
          <h3>Active Users</h3>
          <div class="metric-value">{{ (cumulativeMetrics$ | async)?.activeUsers | number }}</div>
        </div>
      </div>

      <div class="chart-container">
        <canvas #chart [data]="chartData$ | async"></canvas>
      </div>
    </div>
  `
})
export class RealTimeDashboardComponent implements OnInit {
  cumulativeMetrics$!: Observable<CumulativeMetrics>;
  chartData$!: Observable<ChartData>;

  ngOnInit() {
    // Real-time metrics with cumulative calculations
    this.cumulativeMetrics$ = this.metricsService.getMetricsStream().pipe(
      scan((acc, current) => ({
        totalVisits: acc.totalVisits + current.visits,
        totalRevenue: acc.totalRevenue + current.revenue,
        totalConversions: acc.totalConversions + current.conversions,
        totalSessions: acc.totalSessions + current.sessions,
        activeUsers: current.activeUsers, // Current value, not cumulative
        conversionRate: acc.totalSessions > 0 ? acc.totalConversions / acc.totalSessions : 0,
        revenueChange: current.revenue - acc.lastRevenue,
        visitsTrend: this.calculateTrend(current.visits, acc.lastVisits),
        lastRevenue: current.revenue,
        lastVisits: current.visits,
        timestamp: current.timestamp
      }), {
        totalVisits: 0,
        totalRevenue: 0,
        totalConversions: 0,
        totalSessions: 0,
        activeUsers: 0,
        conversionRate: 0,
        revenueChange: 0,
        visitsTrend: 0,
        lastRevenue: 0,
        lastVisits: 0,
        timestamp: Date.now()
      }),
      share()
    );

    // Chart data for visualization
    this.chartData$ = this.cumulativeMetrics$.pipe(
      scan((chartData, metrics) => {
        const newDataPoint = {
          timestamp: metrics.timestamp,
          visits: metrics.totalVisits,
          revenue: metrics.totalRevenue,
          conversions: metrics.totalConversions
        };

        return {
          ...chartData,
          data: [...chartData.data.slice(-50), newDataPoint] // Keep last 50 points
        };
      }, { data: [] as ChartDataPoint[] }),
      map(chartData => this.formatChartData(chartData))
    );
  }

  private calculateTrend(current: number, previous: number): number {
    if (previous === 0) return 0;
    return ((current - previous) / previous) * 100;
  }

  private formatChartData(chartData: any): ChartData {
    return {
      labels: chartData.data.map((point: ChartDataPoint) => 
        new Date(point.timestamp).toLocaleTimeString()
      ),
      datasets: [
        {
          label: 'Visits',
          data: chartData.data.map((point: ChartDataPoint) => point.visits),
          borderColor: 'rgb(75, 192, 192)',
          backgroundColor: 'rgba(75, 192, 192, 0.2)'
        },
        {
          label: 'Revenue',
          data: chartData.data.map((point: ChartDataPoint) => point.revenue),
          borderColor: 'rgb(255, 99, 132)',
          backgroundColor: 'rgba(255, 99, 132, 0.2)'
        }
      ]
    };
  }

  getTrendClass(trend: number): string {
    if (trend > 0) return 'trend-up';
    if (trend < 0) return 'trend-down';
    return 'trend-neutral';
  }
}
```

**Marble Diagram:**
```
Input:  -1-2-3-4-5|
scan((acc, val) => acc + val, 0)
Output: -1-3-6-10-15|

Input:  -1-2-3-4-5|
reduce((acc, val) => acc + val, 0)
Output: -----------15|
```

## Advanced Mathematical Patterns

### 1. Statistical Calculations

```typescript
// Comprehensive statistics service
@Injectable()
export class StatisticsService {
  calculateStatistics<T>(
    data$: Observable<T[]>, 
    valueExtractor: (item: T) => number
  ): Observable<Statistics> {
    return data$.pipe(
      switchMap(data => {
        const values = data.map(valueExtractor);
        
        return forkJoin({
          count: of(values.length),
          sum: from(values).pipe(
            reduce((sum, val) => sum + val, 0)
          ),
          min: from(values).pipe(min()),
          max: from(values).pipe(max()),
          mean: from(values).pipe(
            reduce((sum, val) => sum + val, 0),
            map(sum => values.length > 0 ? sum / values.length : 0)
          ),
          median: of(this.calculateMedian(values)),
          mode: of(this.calculateMode(values)),
          standardDeviation: of(this.calculateStandardDeviation(values))
        });
      })
    );
  }

  private calculateMedian(values: number[]): number {
    const sorted = [...values].sort((a, b) => a - b);
    const mid = Math.floor(sorted.length / 2);
    
    return sorted.length % 2 === 0
      ? (sorted[mid - 1] + sorted[mid]) / 2
      : sorted[mid];
  }

  private calculateMode(values: number[]): number {
    const frequency = values.reduce((freq, val) => {
      freq[val] = (freq[val] || 0) + 1;
      return freq;
    }, {} as Record<number, number>);

    let maxFreq = 0;
    let mode = 0;
    
    Object.entries(frequency).forEach(([val, freq]) => {
      if (freq > maxFreq) {
        maxFreq = freq;
        mode = Number(val);
      }
    });

    return mode;
  }

  private calculateStandardDeviation(values: number[]): number {
    if (values.length === 0) return 0;
    
    const mean = values.reduce((sum, val) => sum + val, 0) / values.length;
    const squaredDiffs = values.map(val => Math.pow(val - mean, 2));
    const avgSquaredDiff = squaredDiffs.reduce((sum, val) => sum + val, 0) / values.length;
    
    return Math.sqrt(avgSquaredDiff);
  }

  // Moving average calculation
  calculateMovingAverage(
    data$: Observable<number>, 
    windowSize: number
  ): Observable<number> {
    return data$.pipe(
      scan((window, value) => {
        window.push(value);
        if (window.length > windowSize) {
          window.shift();
        }
        return window;
      }, [] as number[]),
      filter(window => window.length === windowSize),
      map(window => window.reduce((sum, val) => sum + val, 0) / window.length)
    );
  }

  // Exponential moving average
  calculateEMA(data$: Observable<number>, alpha: number = 0.1): Observable<number> {
    return data$.pipe(
      scan((ema, value) => ema === null ? value : alpha * value + (1 - alpha) * ema, null as number | null),
      filter(ema => ema !== null),
      map(ema => ema!)
    );
  }
}
```

### 2. Performance Metrics

```typescript
// Advanced performance monitoring
@Injectable()
export class PerformanceMetricsService {
  private responseTimeStream$ = new Subject<number>();
  private errorStream$ = new Subject<Error>();
  private throughputStream$ = new Subject<number>();

  // Real-time performance dashboard
  getPerformanceDashboard(): Observable<PerformanceDashboard> {
    return combineLatest([
      this.getResponseTimeMetrics(),
      this.getErrorMetrics(),
      this.getThroughputMetrics(),
      this.getSystemHealthScore()
    ]).pipe(
      map(([responseTime, errors, throughput, healthScore]) => ({
        responseTime,
        errors,
        throughput,
        healthScore,
        timestamp: Date.now()
      }))
    );
  }

  private getResponseTimeMetrics(): Observable<ResponseTimeMetrics> {
    return this.responseTimeStream$.pipe(
      bufferTime(30000), // 30-second windows
      filter(times => times.length > 0),
      switchMap(times => 
        forkJoin({
          current: of(times[times.length - 1]),
          average: from(times).pipe(
            reduce((sum, time) => sum + time, 0),
            map(sum => sum / times.length)
          ),
          min: from(times).pipe(min()),
          max: from(times).pipe(max()),
          p95: of(this.calculatePercentile(times, 95)),
          p99: of(this.calculatePercentile(times, 99)),
          count: of(times.length)
        })
      )
    );
  }

  private getErrorMetrics(): Observable<ErrorMetrics> {
    return this.errorStream$.pipe(
      bufferTime(60000), // 1-minute windows
      map(errors => ({
        count: errors.length,
        rate: errors.length / 60, // errors per second
        types: this.categorizeErrors(errors),
        severity: this.calculateSeverity(errors)
      }))
    );
  }

  private getThroughputMetrics(): Observable<ThroughputMetrics> {
    return this.throughputStream$.pipe(
      bufferTime(10000), // 10-second windows
      switchMap(counts => 
        from(counts).pipe(
          reduce((sum, count) => sum + count, 0),
          map(total => ({
            requestsPerSecond: total / 10,
            total,
            peak: Math.max(...counts),
            average: total / counts.length
          }))
        )
      )
    );
  }

  private getSystemHealthScore(): Observable<number> {
    return combineLatest([
      this.getResponseTimeMetrics(),
      this.getErrorMetrics(),
      this.getThroughputMetrics()
    ]).pipe(
      map(([responseTime, errors, throughput]) => {
        let score = 100;
        
        // Penalize slow response times
        if (responseTime.average > 1000) score -= 20;
        if (responseTime.p95 > 2000) score -= 15;
        
        // Penalize high error rates
        if (errors.rate > 0.1) score -= 30;
        if (errors.rate > 0.05) score -= 15;
        
        // Penalize low throughput
        if (throughput.requestsPerSecond < 10) score -= 10;
        
        return Math.max(0, score);
      })
    );
  }

  private calculatePercentile(values: number[], percentile: number): number {
    const sorted = [...values].sort((a, b) => a - b);
    const index = Math.ceil((percentile / 100) * sorted.length) - 1;
    return sorted[Math.max(0, index)];
  }

  private categorizeErrors(errors: Error[]): Record<string, number> {
    return errors.reduce((categories, error) => {
      const type = this.getErrorType(error);
      categories[type] = (categories[type] || 0) + 1;
      return categories;
    }, {} as Record<string, number>);
  }

  private getErrorType(error: Error): string {
    if (error.message.includes('timeout')) return 'timeout';
    if (error.message.includes('network')) return 'network';
    if (error.message.includes('auth')) return 'authentication';
    return 'unknown';
  }

  private calculateSeverity(errors: Error[]): 'low' | 'medium' | 'high' | 'critical' {
    const errorRate = errors.length / 60; // per second
    
    if (errorRate > 1) return 'critical';
    if (errorRate > 0.1) return 'high';
    if (errorRate > 0.01) return 'medium';
    return 'low';
  }

  // Track metrics
  recordResponseTime(time: number) {
    this.responseTimeStream$.next(time);
  }

  recordError(error: Error) {
    this.errorStream$.next(error);
  }

  recordThroughput(count: number) {
    this.throughputStream$.next(count);
  }
}
```

### 3. Financial Calculations

```typescript
// Financial analytics service
@Injectable()
export class FinancialAnalyticsService {
  constructor(private http: HttpClient) {}

  // Portfolio analysis
  analyzePortfolio(portfolioId: string): Observable<PortfolioAnalysis> {
    return this.http.get<Investment[]>(`/api/portfolio/${portfolioId}`).pipe(
      switchMap(investments => 
        forkJoin({
          totalValue: from(investments).pipe(
            reduce((total, investment) => total + investment.currentValue, 0)
          ),
          totalCost: from(investments).pipe(
            reduce((total, investment) => total + investment.costBasis, 0)
          ),
          bestPerformer: from(investments).pipe(
            max((a, b) => a.returnPercentage < b.returnPercentage ? -1 : 1)
          ),
          worstPerformer: from(investments).pipe(
            min((a, b) => a.returnPercentage < b.returnPercentage ? -1 : 1)
          ),
          averageReturn: from(investments).pipe(
            reduce((sum, investment) => sum + investment.returnPercentage, 0),
            map(sum => investments.length > 0 ? sum / investments.length : 0)
          ),
          riskScore: of(this.calculateRiskScore(investments)),
          diversificationScore: of(this.calculateDiversification(investments))
        })
      ),
      map(analysis => ({
        ...analysis,
        totalReturn: analysis.totalValue - analysis.totalCost,
        returnPercentage: analysis.totalCost > 0 ? 
          ((analysis.totalValue - analysis.totalCost) / analysis.totalCost) * 100 : 0
      }))
    );
  }

  // Real-time P&L calculation
  getRealTimePnL(positions: Position[]): Observable<PnLCalculation> {
    return interval(1000).pipe(
      switchMap(() => this.getCurrentPrices(positions.map(p => p.symbol))),
      map(prices => {
        const calculations = positions.map(position => {
          const currentPrice = prices[position.symbol];
          const currentValue = position.quantity * currentPrice;
          const costBasis = position.quantity * position.averagePrice;
          const unrealizedPnL = currentValue - costBasis;
          
          return {
            symbol: position.symbol,
            quantity: position.quantity,
            averagePrice: position.averagePrice,
            currentPrice,
            currentValue,
            costBasis,
            unrealizedPnL,
            unrealizedPnLPercentage: costBasis > 0 ? (unrealizedPnL / costBasis) * 100 : 0
          };
        });

        return {
          positions: calculations,
          totalUnrealizedPnL: calculations.reduce((sum, calc) => sum + calc.unrealizedPnL, 0),
          totalValue: calculations.reduce((sum, calc) => sum + calc.currentValue, 0),
          totalCostBasis: calculations.reduce((sum, calc) => sum + calc.costBasis, 0)
        };
      })
    );
  }

  private calculateRiskScore(investments: Investment[]): number {
    // Simple risk calculation based on volatility and concentration
    const volatilities = investments.map(inv => inv.volatility || 0);
    const avgVolatility = volatilities.reduce((sum, vol) => sum + vol, 0) / volatilities.length;
    
    // Check concentration risk
    const totalValue = investments.reduce((sum, inv) => sum + inv.currentValue, 0);
    const maxPosition = Math.max(...investments.map(inv => inv.currentValue));
    const concentrationRisk = maxPosition / totalValue;
    
    return Math.min(100, avgVolatility * 100 + concentrationRisk * 50);
  }

  private calculateDiversification(investments: Investment[]): number {
    const sectors = [...new Set(investments.map(inv => inv.sector))];
    const sectorCount = sectors.length;
    const totalInvestments = investments.length;
    
    // More sectors = better diversification
    return Math.min(100, (sectorCount / totalInvestments) * 100);
  }

  private getCurrentPrices(symbols: string[]): Observable<Record<string, number>> {
    return this.http.get<Record<string, number>>(`/api/prices?symbols=${symbols.join(',')}`);
  }
}
```

## Best Practices

### 1. Performance Optimization

```typescript
// ✅ Use shareReplay for expensive calculations
const expensiveStats$ = data$.pipe(
  switchMap(data => this.calculateComplexStatistics(data)),
  shareReplay(1)
);

// ✅ Buffer data for batch processing
const batchedMetrics$ = metricsStream$.pipe(
  bufferTime(5000),
  filter(batch => batch.length > 0),
  map(batch => this.processBatch(batch))
);

// ✅ Use scan for running calculations
const runningAverage$ = values$.pipe(
  scan((acc, value) => ({
    sum: acc.sum + value,
    count: acc.count + 1,
    average: (acc.sum + value) / (acc.count + 1)
  }), { sum: 0, count: 0, average: 0 })
);
```

### 2. Memory Management

```typescript
// ✅ Limit window sizes for moving calculations
const movingAverage$ = values$.pipe(
  scan((window, value) => {
    window.push(value);
    if (window.length > 100) { // Limit window size
      window.shift();
    }
    return window;
  }, [] as number[]),
  map(window => window.reduce((sum, val) => sum + val, 0) / window.length)
);

// ✅ Use takeUntil for cleanup
@Component({})
export class StatsComponent implements OnDestroy {
  private destroy$ = new Subject<void>();
  
  stats$ = this.dataService.getData().pipe(
    scan((acc, data) => this.updateStats(acc, data), initialStats),
    takeUntil(this.destroy$)
  );

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

### 3. Error Handling

```typescript
// ✅ Handle division by zero and edge cases
const safeAverage$ = values$.pipe(
  reduce((sum, val) => sum + val, 0),
  map(sum => count > 0 ? sum / count : 0),
  catchError(error => {
    console.error('Average calculation failed:', error);
    return of(0);
  })
);

// ✅ Validate input data
const validatedStats$ = data$.pipe(
  map(data => data.filter(item => this.isValidNumber(item.value))),
  switchMap(validData => this.calculateStatistics(validData))
);
```

## Common Pitfalls

### 1. Memory Leaks with Unbounded Accumulation

```typescript
// ❌ Unbounded accumulation
const problematic$ = values$.pipe(
  scan((acc, value) => [...acc, value], []) // Grows forever
);

// ✅ Bounded accumulation
const safe$ = values$.pipe(
  scan((acc, value) => {
    acc.push(value);
    return acc.slice(-1000); // Keep only last 1000 values
  }, [] as number[])
);
```

### 2. Incorrect Mathematical Operations

```typescript
// ❌ Not handling empty streams
const risky$ = emptyStream$.pipe(
  reduce((sum, val) => sum + val, 0) // Will never emit
);

// ✅ Handle empty streams
const safe$ = stream$.pipe(
  reduce((sum, val) => sum + val, 0),
  defaultIfEmpty(0)
);
```

## Exercises

### Exercise 1: Real-time Analytics Dashboard
Create a comprehensive analytics dashboard that calculates and displays real-time statistics including moving averages, percentiles, and trend analysis.

### Exercise 2: Financial Portfolio Tracker
Build a portfolio tracking system that calculates real-time P&L, risk metrics, and performance statistics with proper error handling and optimization.

### Exercise 3: Performance Monitoring System
Implement a system performance monitoring solution that tracks and analyzes various metrics with statistical calculations and alerting.

## Summary

Mathematical operators provide essential tools for data analysis and computation in reactive streams:

- **count()**: Count emissions with optional predicates
- **max()/min()**: Find extreme values with custom comparisons
- **reduce()**: Accumulate final results
- **scan()**: Track intermediate accumulations
- Statistical functions for comprehensive data analysis

These operators enable sophisticated mathematical operations while maintaining reactive principles, making them ideal for analytics, monitoring, and financial applications.

## Next Steps

In the next lesson, we'll explore **Multicasting Operators**, which provide advanced sharing and broadcasting capabilities for Observable streams.
