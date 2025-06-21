# Drag & Drop Patterns with RxJS

## Introduction

This lesson explores sophisticated drag and drop patterns using RxJS observables. We'll build reusable, performant drag and drop systems that handle complex scenarios like multi-touch, constraints, animations, and data transfer.

## Learning Objectives

- Build reactive drag and drop systems
- Handle complex interaction patterns
- Implement drag constraints and validation
- Create smooth animations with observables
- Build sortable lists and kanban boards
- Handle file uploads via drag and drop

## Core Drag & Drop Observable Pattern

### Basic Draggable Element

```typescript
// drag.service.ts
import { Injectable, ElementRef } from '@angular/core';
import { fromEvent, merge, EMPTY } from 'rxjs';
import { map, switchMap, takeUntil, startWith, tap } from 'rxjs/operators';

export interface DragEvent {
  element: HTMLElement;
  startPosition: { x: number; y: number };
  currentPosition: { x: number; y: number };
  offset: { x: number; y: number };
  isDragging: boolean;
}

@Injectable({
  providedIn: 'root'
})
export class DragService {
  
  createDraggable(element: HTMLElement) {
    const mouseDown$ = fromEvent<MouseEvent>(element, 'mousedown');
    const mouseMove$ = fromEvent<MouseEvent>(document, 'mousemove');
    const mouseUp$ = fromEvent<MouseEvent>(document, 'mouseup');
    
    const touchStart$ = fromEvent<TouchEvent>(element, 'touchstart');
    const touchMove$ = fromEvent<TouchEvent>(document, 'touchmove');
    const touchEnd$ = fromEvent<TouchEvent>(document, 'touchend');
    
    // Normalize touch and mouse events
    const start$ = merge(
      mouseDown$.pipe(map(e => ({ x: e.clientX, y: e.clientY, event: e }))),
      touchStart$.pipe(map(e => ({ 
        x: e.touches[0].clientX, 
        y: e.touches[0].clientY, 
        event: e 
      })))
    );
    
    const move$ = merge(
      mouseMove$.pipe(map(e => ({ x: e.clientX, y: e.clientY }))),
      touchMove$.pipe(map(e => ({ 
        x: e.touches[0].clientX, 
        y: e.touches[0].clientY 
      })))
    );
    
    const end$ = merge(mouseUp$, touchEnd$);
    
    return start$.pipe(
      tap(e => e.event.preventDefault()),
      map(startEvent => {
        const startPosition = { x: startEvent.x, y: startEvent.y };
        const rect = element.getBoundingClientRect();
        const offset = {
          x: startEvent.x - rect.left,
          y: startEvent.y - rect.top
        };
        
        return move$.pipe(
          startWith(startEvent),
          map(moveEvent => ({
            element,
            startPosition,
            currentPosition: { x: moveEvent.x, y: moveEvent.y },
            offset,
            isDragging: true
          } as DragEvent)),
          takeUntil(end$)
        );
      }),
      switchMap(drag$ => drag$)
    );
  }
}
```

### Advanced Draggable Directive

```typescript
// draggable.directive.ts
import { Directive, ElementRef, Input, Output, EventEmitter, OnInit, OnDestroy } from '@angular/core';
import { Subject, combineLatest } from 'rxjs';
import { takeUntil, throttleTime, distinctUntilChanged } from 'rxjs/operators';
import { DragService, DragEvent } from './drag.service';

export interface DragConstraints {
  boundary?: HTMLElement;
  axis?: 'x' | 'y' | 'both';
  grid?: { x: number; y: number };
  minPosition?: { x: number; y: number };
  maxPosition?: { x: number; y: number };
}

@Directive({
  selector: '[appDraggable]'
})
export class DraggableDirective implements OnInit, OnDestroy {
  @Input() dragConstraints: DragConstraints = {};
  @Input() dragData: any;
  @Input() dragDisabled = false;
  @Input() dragThrottle = 16; // ~60fps
  
  @Output() dragStart = new EventEmitter<DragEvent>();
  @Output() dragMove = new EventEmitter<DragEvent>();
  @Output() dragEnd = new EventEmitter<DragEvent>();
  
  private destroy$ = new Subject<void>();
  private originalPosition = { x: 0, y: 0 };
  
  constructor(
    private elementRef: ElementRef<HTMLElement>,
    private dragService: DragService
  ) {}
  
  ngOnInit() {
    this.setupDragBehavior();
  }
  
  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
  
  private setupDragBehavior() {
    if (this.dragDisabled) return;
    
    const element = this.elementRef.nativeElement;
    element.style.position = 'relative';
    
    const drag$ = this.dragService.createDraggable(element);
    
    drag$.pipe(
      throttleTime(this.dragThrottle),
      distinctUntilChanged((a, b) => 
        a.currentPosition.x === b.currentPosition.x && 
        a.currentPosition.y === b.currentPosition.y
      ),
      takeUntil(this.destroy$)
    ).subscribe({
      next: (dragEvent) => this.handleDrag(dragEvent),
      complete: () => this.handleDragEnd()
    });
  }
  
  private handleDrag(dragEvent: DragEvent) {
    const constrainedPosition = this.applyConstraints(dragEvent);
    
    // Apply visual transformation
    const element = this.elementRef.nativeElement;
    element.style.transform = `translate(${constrainedPosition.x}px, ${constrainedPosition.y}px)`;
    element.classList.add('dragging');
    
    const constrainedEvent = { ...dragEvent, currentPosition: constrainedPosition };
    
    if (this.isFirstDrag(dragEvent)) {
      this.dragStart.emit(constrainedEvent);
    } else {
      this.dragMove.emit(constrainedEvent);
    }
  }
  
  private handleDragEnd() {
    const element = this.elementRef.nativeElement;
    element.classList.remove('dragging');
    this.dragEnd.emit();
  }
  
  private applyConstraints(dragEvent: DragEvent): { x: number; y: number } {
    const { currentPosition, startPosition } = dragEvent;
    let newX = currentPosition.x - startPosition.x;
    let newY = currentPosition.y - startPosition.y;
    
    // Apply axis constraints
    if (this.dragConstraints.axis === 'x') newY = 0;
    if (this.dragConstraints.axis === 'y') newX = 0;
    
    // Apply grid snapping
    if (this.dragConstraints.grid) {
      newX = Math.round(newX / this.dragConstraints.grid.x) * this.dragConstraints.grid.x;
      newY = Math.round(newY / this.dragConstraints.grid.y) * this.dragConstraints.grid.y;
    }
    
    // Apply boundary constraints
    if (this.dragConstraints.boundary) {
      const bounds = this.getBoundaryLimits();
      newX = Math.max(bounds.minX, Math.min(bounds.maxX, newX));
      newY = Math.max(bounds.minY, Math.min(bounds.maxY, newY));
    }
    
    // Apply min/max position constraints
    if (this.dragConstraints.minPosition) {
      newX = Math.max(this.dragConstraints.minPosition.x, newX);
      newY = Math.max(this.dragConstraints.minPosition.y, newY);
    }
    
    if (this.dragConstraints.maxPosition) {
      newX = Math.min(this.dragConstraints.maxPosition.x, newX);
      newY = Math.min(this.dragConstraints.maxPosition.y, newY);
    }
    
    return { x: newX, y: newY };
  }
  
  private getBoundaryLimits() {
    const element = this.elementRef.nativeElement;
    const boundary = this.dragConstraints.boundary!;
    const elementRect = element.getBoundingClientRect();
    const boundaryRect = boundary.getBoundingClientRect();
    
    return {
      minX: boundaryRect.left - elementRect.left,
      maxX: boundaryRect.right - elementRect.right,
      minY: boundaryRect.top - elementRect.top,
      maxY: boundaryRect.bottom - elementRect.bottom
    };
  }
  
  private isFirstDrag(dragEvent: DragEvent): boolean {
    return dragEvent.currentPosition.x === dragEvent.startPosition.x &&
           dragEvent.currentPosition.y === dragEvent.startPosition.y;
  }
}
```

## Drop Zone Implementation

### Drop Zone Service

```typescript
// drop-zone.service.ts
import { Injectable } from '@angular/core';
import { Subject, Observable, fromEvent, merge } from 'rxjs';
import { filter, map, shareReplay } from 'rxjs/operators';

export interface DropEvent {
  dropZone: HTMLElement;
  dragData: any;
  position: { x: number; y: number };
  dropAllowed: boolean;
}

export interface DropZoneConfig {
  acceptedTypes?: string[];
  acceptedData?: (data: any) => boolean;
  multiDrop?: boolean;
  autoScroll?: boolean;
}

@Injectable({
  providedIn: 'root'
})
export class DropZoneService {
  private dropEvents$ = new Subject<DropEvent>();
  private activeDropZones = new Map<HTMLElement, DropZoneConfig>();
  
  getDropEvents(): Observable<DropEvent> {
    return this.dropEvents$.asObservable();
  }
  
  registerDropZone(element: HTMLElement, config: DropZoneConfig = {}) {
    this.activeDropZones.set(element, config);
    
    const dragOver$ = fromEvent<DragEvent>(element, 'dragover');
    const dragEnter$ = fromEvent<DragEvent>(element, 'dragenter');
    const dragLeave$ = fromEvent<DragEvent>(element, 'dragleave');
    const drop$ = fromEvent<DragEvent>(element, 'drop');
    
    // Handle native HTML5 drag and drop
    merge(dragOver$, dragEnter$).subscribe(e => {
      e.preventDefault();
      e.stopPropagation();
      element.classList.add('drag-over');
    });
    
    dragLeave$.subscribe(e => {
      if (!element.contains(e.relatedTarget as Node)) {
        element.classList.remove('drag-over');
      }
    });
    
    drop$.subscribe(e => {
      e.preventDefault();
      e.stopPropagation();
      element.classList.remove('drag-over');
      
      const dragData = this.extractDragData(e);
      if (this.canAcceptDrop(element, dragData)) {
        this.dropEvents$.next({
          dropZone: element,
          dragData,
          position: { x: e.clientX, y: e.clientY },
          dropAllowed: true
        });
      }
    });
  }
  
  unregisterDropZone(element: HTMLElement) {
    this.activeDropZones.delete(element);
  }
  
  private extractDragData(event: DragEvent): any {
    try {
      const jsonData = event.dataTransfer?.getData('application/json');
      return jsonData ? JSON.parse(jsonData) : null;
    } catch {
      return event.dataTransfer?.getData('text/plain') || null;
    }
  }
  
  private canAcceptDrop(element: HTMLElement, dragData: any): boolean {
    const config = this.activeDropZones.get(element);
    if (!config) return false;
    
    if (config.acceptedData) {
      return config.acceptedData(dragData);
    }
    
    return true;
  }
}
```

### Drop Zone Directive

```typescript
// drop-zone.directive.ts
import { Directive, ElementRef, Input, Output, EventEmitter, OnInit, OnDestroy } from '@angular/core';
import { DropZoneService, DropEvent, DropZoneConfig } from './drop-zone.service';
import { Subject } from 'rxjs';
import { takeUntil, filter } from 'rxjs/operators';

@Directive({
  selector: '[appDropZone]'
})
export class DropZoneDirective implements OnInit, OnDestroy {
  @Input() dropZoneConfig: DropZoneConfig = {};
  @Input() dropZoneId: string = '';
  
  @Output() itemDropped = new EventEmitter<DropEvent>();
  @Output() dragEnter = new EventEmitter<DragEvent>();
  @Output() dragLeave = new EventEmitter<DragEvent>();
  
  private destroy$ = new Subject<void>();
  
  constructor(
    private elementRef: ElementRef<HTMLElement>,
    private dropZoneService: DropZoneService
  ) {}
  
  ngOnInit() {
    const element = this.elementRef.nativeElement;
    element.classList.add('drop-zone');
    
    this.dropZoneService.registerDropZone(element, this.dropZoneConfig);
    
    this.dropZoneService.getDropEvents().pipe(
      filter(event => event.dropZone === element),
      takeUntil(this.destroy$)
    ).subscribe(dropEvent => {
      this.itemDropped.emit(dropEvent);
    });
  }
  
  ngOnDestroy() {
    this.dropZoneService.unregisterDropZone(this.elementRef.nativeElement);
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

## Sortable List Implementation

### Sortable List Component

```typescript
// sortable-list.component.ts
import { Component, Input, Output, EventEmitter, ChangeDetectionStrategy } from '@angular/core';
import { BehaviorSubject, combineLatest, Observable } from 'rxjs';
import { map, distinctUntilChanged } from 'rxjs/operators';

export interface SortableItem {
  id: string;
  data: any;
  position: number;
}

export interface SortEvent {
  item: SortableItem;
  previousIndex: number;
  currentIndex: number;
  items: SortableItem[];
}

@Component({
  selector: 'app-sortable-list',
  template: `
    <div class="sortable-container" 
         [class.sorting]="isDragging$ | async">
      <div *ngFor="let item of sortedItems$ | async; trackBy: trackByFn"
           class="sortable-item"
           [class.dragging]="item.id === draggingItemId"
           [class.placeholder]="item.id === placeholderId"
           appDraggable
           appDropZone
           [dragData]="item"
           [dropZoneConfig]="dropConfig"
           (dragStart)="onDragStart(item, $event)"
           (dragMove)="onDragMove($event)"
           (dragEnd)="onDragEnd()"
           (itemDropped)="onItemDropped(item, $event)">
        
        <div class="item-content">
          <ng-content [ngTemplateOutlet]="itemTemplate" 
                     [ngTemplateOutletContext]="{ item: item.data, index: item.position }">
          </ng-content>
        </div>
        
        <div class="drag-handle" *ngIf="showDragHandle">
          <svg width="16" height="16" viewBox="0 0 16 16">
            <path d="M2 4h12v2H2V4zm0 6h12v2H2v-2z"/>
          </svg>
        </div>
      </div>
    </div>
  `,
  styleUrls: ['./sortable-list.component.scss'],
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class SortableListComponent {
  @Input() items: any[] = [];
  @Input() itemTemplate: any;
  @Input() showDragHandle = true;
  @Input() animation = true;
  
  @Output() itemsChanged = new EventEmitter<any[]>();
  @Output() sortChanged = new EventEmitter<SortEvent>();
  
  private items$ = new BehaviorSubject<SortableItem[]>([]);
  private draggingItem$ = new BehaviorSubject<SortableItem | null>(null);
  private placeholderPosition$ = new BehaviorSubject<number>(-1);
  
  sortedItems$: Observable<SortableItem[]>;
  isDragging$: Observable<boolean>;
  draggingItemId = '';
  placeholderId = '';
  
  dropConfig = {
    acceptedData: (data: any) => !!data?.id
  };
  
  constructor() {
    this.sortedItems$ = combineLatest([
      this.items$,
      this.draggingItem$,
      this.placeholderPosition$
    ]).pipe(
      map(([items, draggingItem, placeholderPos]) => 
        this.calculateSortedItems(items, draggingItem, placeholderPos)
      ),
      distinctUntilChanged()
    );
    
    this.isDragging$ = this.draggingItem$.pipe(
      map(item => !!item),
      distinctUntilChanged()
    );
  }
  
  ngOnChanges() {
    const sortableItems = this.items.map((item, index) => ({
      id: item.id || `item-${index}`,
      data: item,
      position: index
    }));
    this.items$.next(sortableItems);
  }
  
  onDragStart(item: SortableItem, event: any) {
    this.draggingItem$.next(item);
    this.draggingItemId = item.id;
    this.placeholderId = `placeholder-${item.id}`;
  }
  
  onDragMove(event: any) {
    const newPosition = this.calculateDropPosition(event.currentPosition);
    this.placeholderPosition$.next(newPosition);
  }
  
  onDragEnd() {
    const draggingItem = this.draggingItem$.value;
    const newPosition = this.placeholderPosition$.value;
    
    if (draggingItem && newPosition >= 0) {
      this.performSort(draggingItem, newPosition);
    }
    
    this.draggingItem$.next(null);
    this.placeholderPosition$.next(-1);
    this.draggingItemId = '';
    this.placeholderId = '';
  }
  
  onItemDropped(targetItem: SortableItem, event: any) {
    const droppedItem = event.dragData as SortableItem;
    if (droppedItem && droppedItem.id !== targetItem.id) {
      this.performSort(droppedItem, targetItem.position);
    }
  }
  
  private calculateSortedItems(
    items: SortableItem[], 
    draggingItem: SortableItem | null, 
    placeholderPos: number
  ): SortableItem[] {
    if (!draggingItem || placeholderPos < 0) {
      return items;
    }
    
    const filteredItems = items.filter(item => item.id !== draggingItem.id);
    const result = [...filteredItems];
    
    // Insert placeholder
    const placeholder: SortableItem = {
      id: this.placeholderId,
      data: { ...draggingItem.data, isPlaceholder: true },
      position: placeholderPos
    };
    
    result.splice(placeholderPos, 0, placeholder);
    
    return result.map((item, index) => ({
      ...item,
      position: index
    }));
  }
  
  private calculateDropPosition(mousePosition: { x: number; y: number }): number {
    const items = this.items$.value;
    // Implementation would calculate based on mouse position relative to items
    // This is a simplified version
    return Math.floor(items.length / 2);
  }
  
  private performSort(item: SortableItem, newPosition: number) {
    const items = this.items$.value;
    const oldPosition = item.position;
    
    if (oldPosition === newPosition) return;
    
    // Create new sorted array
    const newItems = [...items];
    const [movedItem] = newItems.splice(oldPosition, 1);
    newItems.splice(newPosition, 0, movedItem);
    
    // Update positions
    const updatedItems = newItems.map((item, index) => ({
      ...item,
      position: index
    }));
    
    this.items$.next(updatedItems);
    
    // Emit events
    const sortEvent: SortEvent = {
      item: movedItem,
      previousIndex: oldPosition,
      currentIndex: newPosition,
      items: updatedItems
    };
    
    this.sortChanged.emit(sortEvent);
    this.itemsChanged.emit(updatedItems.map(i => i.data));
  }
  
  trackByFn(index: number, item: SortableItem): string {
    return item.id;
  }
}
```

## Kanban Board Implementation

### Kanban Board Component

```typescript
// kanban-board.component.ts
import { Component, Input, Output, EventEmitter } from '@angular/core';
import { BehaviorSubject, Observable } from 'rxjs';
import { map } from 'rxjs/operators';

export interface KanbanColumn {
  id: string;
  title: string;
  items: KanbanItem[];
  maxItems?: number;
  acceptedTypes?: string[];
}

export interface KanbanItem {
  id: string;
  title: string;
  type?: string;
  data: any;
}

export interface KanbanMoveEvent {
  item: KanbanItem;
  fromColumn: string;
  toColumn: string;
  fromIndex: number;
  toIndex: number;
}

@Component({
  selector: 'app-kanban-board',
  template: `
    <div class="kanban-board">
      <div class="kanban-column" 
           *ngFor="let column of columns$ | async; trackBy: trackColumn">
        
        <div class="column-header">
          <h3>{{ column.title }}</h3>
          <span class="item-count">{{ column.items.length }}</span>
          <span *ngIf="column.maxItems" class="max-items">/ {{ column.maxItems }}</span>
        </div>
        
        <div class="column-content"
             appDropZone
             [dropZoneConfig]="getDropConfig(column)"
             (itemDropped)="onItemDropped(column, $event)">
          
          <div *ngFor="let item of column.items; trackBy: trackItem; let i = index"
               class="kanban-item"
               [class.dragging]="item.id === draggingItemId"
               appDraggable
               [dragData]="{ item, columnId: column.id, index: i }"
               (dragStart)="onDragStart(item)"
               (dragEnd)="onDragEnd()">
            
            <div class="item-header">
              <span class="item-title">{{ item.title }}</span>
              <span class="item-type" *ngIf="item.type">{{ item.type }}</span>
            </div>
            
            <div class="item-content">
              <ng-content [ngTemplateOutlet]="itemTemplate"
                         [ngTemplateOutletContext]="{ item: item, column: column }">
              </ng-content>
            </div>
          </div>
          
          <div class="drop-placeholder" 
               *ngIf="isValidDropTarget(column)"
               [class.visible]="showDropPlaceholder">
            Drop item here
          </div>
        </div>
      </div>
    </div>
  `,
  styleUrls: ['./kanban-board.component.scss']
})
export class KanbanBoardComponent {
  @Input() columns: KanbanColumn[] = [];
  @Input() itemTemplate: any;
  @Input() allowCrossColumnDrop = true;
  
  @Output() itemMoved = new EventEmitter<KanbanMoveEvent>();
  @Output() columnChanged = new EventEmitter<KanbanColumn>();
  
  private columns$ = new BehaviorSubject<KanbanColumn[]>([]);
  draggingItemId = '';
  showDropPlaceholder = false;
  
  ngOnChanges() {
    this.columns$.next(this.columns);
  }
  
  getDropConfig(column: KanbanColumn) {
    return {
      acceptedData: (data: any) => {
        if (!data?.item) return false;
        
        // Check if column has space
        if (column.maxItems && column.items.length >= column.maxItems) {
          return false;
        }
        
        // Check accepted types
        if (column.acceptedTypes && data.item.type) {
          return column.acceptedTypes.includes(data.item.type);
        }
        
        return this.allowCrossColumnDrop || data.columnId === column.id;
      }
    };
  }
  
  onDragStart(item: KanbanItem) {
    this.draggingItemId = item.id;
    this.showDropPlaceholder = true;
  }
  
  onDragEnd() {
    this.draggingItemId = '';
    this.showDropPlaceholder = false;
  }
  
  onItemDropped(targetColumn: KanbanColumn, event: any) {
    const { item, columnId: fromColumnId, index: fromIndex } = event.dragData;
    
    if (fromColumnId === targetColumn.id) {
      // Same column reordering
      this.reorderInColumn(targetColumn, fromIndex, event.position);
    } else {
      // Cross-column move
      this.moveItemBetweenColumns(item, fromColumnId, targetColumn.id, fromIndex);
    }
  }
  
  private moveItemBetweenColumns(
    item: KanbanItem, 
    fromColumnId: string, 
    toColumnId: string, 
    fromIndex: number
  ) {
    const columns = this.columns$.value.map(col => ({ ...col, items: [...col.items] }));
    const fromColumn = columns.find(col => col.id === fromColumnId);
    const toColumn = columns.find(col => col.id === toColumnId);
    
    if (!fromColumn || !toColumn) return;
    
    // Remove from source column
    const [movedItem] = fromColumn.items.splice(fromIndex, 1);
    
    // Add to target column
    const toIndex = toColumn.items.length;
    toColumn.items.push(movedItem);
    
    this.columns$.next(columns);
    
    const moveEvent: KanbanMoveEvent = {
      item: movedItem,
      fromColumn: fromColumnId,
      toColumn: toColumnId,
      fromIndex,
      toIndex
    };
    
    this.itemMoved.emit(moveEvent);
    this.columnChanged.emit(toColumn);
  }
  
  private reorderInColumn(column: KanbanColumn, fromIndex: number, dropPosition: any) {
    // Implementation for same-column reordering
    const newIndex = this.calculateDropIndex(column, dropPosition);
    if (fromIndex === newIndex) return;
    
    const items = [...column.items];
    const [movedItem] = items.splice(fromIndex, 1);
    items.splice(newIndex, 0, movedItem);
    
    const updatedColumn = { ...column, items };
    this.updateColumn(updatedColumn);
  }
  
  private calculateDropIndex(column: KanbanColumn, position: any): number {
    // Calculate drop index based on mouse position
    return Math.max(0, Math.min(column.items.length, position.index || 0));
  }
  
  private updateColumn(column: KanbanColumn) {
    const columns = this.columns$.value.map(col => 
      col.id === column.id ? column : col
    );
    this.columns$.next(columns);
    this.columnChanged.emit(column);
  }
  
  isValidDropTarget(column: KanbanColumn): boolean {
    if (!this.draggingItemId) return false;
    if (column.maxItems && column.items.length >= column.maxItems) return false;
    return true;
  }
  
  trackColumn(index: number, column: KanbanColumn): string {
    return column.id;
  }
  
  trackItem(index: number, item: KanbanItem): string {
    return item.id;
  }
}
```

## File Upload Drag & Drop

### File Drop Zone Component

```typescript
// file-drop-zone.component.ts
import { Component, Input, Output, EventEmitter, ChangeDetectionStrategy } from '@angular/core';
import { fromEvent, merge, Subject } from 'rxjs';
import { takeUntil, map, filter } from 'rxjs/operators';

export interface FileDropEvent {
  files: File[];
  position: { x: number; y: number };
  dataTransfer: DataTransfer;
}

export interface FileUploadProgress {
  file: File;
  progress: number;
  status: 'pending' | 'uploading' | 'completed' | 'error';
  error?: string;
}

@Component({
  selector: 'app-file-drop-zone',
  template: `
    <div class="file-drop-zone"
         #dropZone
         [class.drag-over]="isDragOver"
         [class.disabled]="disabled">
      
      <div class="drop-area" *ngIf="!files.length">
        <div class="drop-icon">
          <svg width="48" height="48" viewBox="0 0 48 48">
            <path d="M14 2H6a2 2 0 0 0-2 2v40a2 2 0 0 0 2 2h36a2 2 0 0 0 2-2V8l-6-6z"/>
            <polyline points="14,2 14,8 20,8"/>
            <line x1="16" y1="13" x2="8" y2="21"/>
            <line x1="8" y1="13" x2="16" y2="21"/>
          </svg>
        </div>
        
        <div class="drop-text">
          <p class="primary-text">{{ primaryText }}</p>
          <p class="secondary-text">{{ secondaryText }}</p>
        </div>
        
        <button type="button" 
                class="browse-button"
                (click)="triggerFileInput()"
                [disabled]="disabled">
          {{ browseButtonText }}
        </button>
      </div>
      
      <div class="file-list" *ngIf="files.length">
        <div class="file-item" 
             *ngFor="let file of uploadProgress; trackBy: trackFile">
          
          <div class="file-info">
            <div class="file-name">{{ file.file.name }}</div>
            <div class="file-size">{{ formatFileSize(file.file.size) }}</div>
          </div>
          
          <div class="file-progress">
            <div class="progress-bar">
              <div class="progress-fill" 
                   [style.width.%]="file.progress">
              </div>
            </div>
            <span class="progress-text">{{ file.progress }}%</span>
          </div>
          
          <div class="file-status">
            <span [ngSwitch]="file.status" class="status-icon">
              <span *ngSwitchCase="'pending'">⏳</span>
              <span *ngSwitchCase="'uploading'">⬆️</span>
              <span *ngSwitchCase="'completed'">✅</span>
              <span *ngSwitchCase="'error'">❌</span>
            </span>
          </div>
          
          <button type="button" 
                  class="remove-button"
                  (click)="removeFile(file.file)"
                  [disabled]="file.status === 'uploading'">
            ×
          </button>
        </div>
      </div>
      
      <input type="file"
             #fileInput
             [multiple]="multiple"
             [accept]="acceptedTypes"
             (change)="onFileInputChange($event)"
             style="display: none;">
    </div>
  `,
  styleUrls: ['./file-drop-zone.component.scss'],
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class FileDropZoneComponent implements OnInit, OnDestroy {
  @Input() multiple = true;
  @Input() acceptedTypes = '';
  @Input() maxFileSize = 10 * 1024 * 1024; // 10MB
  @Input() maxFiles = 10;
  @Input() disabled = false;
  @Input() primaryText = 'Drag & drop files here';
  @Input() secondaryText = 'or click to browse';
  @Input() browseButtonText = 'Browse Files';
  @Input() autoUpload = false;
  
  @Output() filesDropped = new EventEmitter<FileDropEvent>();
  @Output() filesSelected = new EventEmitter<File[]>();
  @Output() uploadProgress = new EventEmitter<FileUploadProgress[]>();
  @Output() uploadComplete = new EventEmitter<File[]>();
  @Output() uploadError = new EventEmitter<{ file: File; error: string }>();
  
  files: File[] = [];
  uploadProgress: FileUploadProgress[] = [];
  isDragOver = false;
  
  private destroy$ = new Subject<void>();
  
  @ViewChild('dropZone', { static: true }) dropZone!: ElementRef<HTMLElement>;
  @ViewChild('fileInput', { static: true }) fileInput!: ElementRef<HTMLInputElement>;
  
  ngOnInit() {
    this.setupDropListeners();
  }
  
  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
  
  private setupDropListeners() {
    const element = this.dropZone.nativeElement;
    
    const dragEnter$ = fromEvent<DragEvent>(element, 'dragenter');
    const dragOver$ = fromEvent<DragEvent>(element, 'dragover');
    const dragLeave$ = fromEvent<DragEvent>(element, 'dragleave');
    const drop$ = fromEvent<DragEvent>(element, 'drop');
    
    // Prevent default behavior
    merge(dragEnter$, dragOver$, drop$).pipe(
      takeUntil(this.destroy$)
    ).subscribe(e => {
      e.preventDefault();
      e.stopPropagation();
    });
    
    // Handle drag enter/over
    merge(dragEnter$, dragOver$).pipe(
      filter(() => !this.disabled),
      takeUntil(this.destroy$)
    ).subscribe(e => {
      this.isDragOver = true;
      e.dataTransfer!.dropEffect = 'copy';
    });
    
    // Handle drag leave
    dragLeave$.pipe(
      filter(e => !element.contains(e.relatedTarget as Node)),
      takeUntil(this.destroy$)
    ).subscribe(() => {
      this.isDragOver = false;
    });
    
    // Handle drop
    drop$.pipe(
      filter(() => !this.disabled),
      takeUntil(this.destroy$)
    ).subscribe(e => {
      this.isDragOver = false;
      this.handleFileDrop(e);
    });
  }
  
  private handleFileDrop(event: DragEvent) {
    const files = Array.from(event.dataTransfer?.files || []);
    const validFiles = this.validateFiles(files);
    
    if (validFiles.length > 0) {
      this.addFiles(validFiles);
      
      const dropEvent: FileDropEvent = {
        files: validFiles,
        position: { x: event.clientX, y: event.clientY },
        dataTransfer: event.dataTransfer!
      };
      
      this.filesDropped.emit(dropEvent);
    }
  }
  
  onFileInputChange(event: Event) {
    const input = event.target as HTMLInputElement;
    const files = Array.from(input.files || []);
    const validFiles = this.validateFiles(files);
    
    if (validFiles.length > 0) {
      this.addFiles(validFiles);
      this.filesSelected.emit(validFiles);
    }
    
    // Reset input
    input.value = '';
  }
  
  triggerFileInput() {
    if (!this.disabled) {
      this.fileInput.nativeElement.click();
    }
  }
  
  private validateFiles(files: File[]): File[] {
    return files.filter(file => {
      // Check file size
      if (file.size > this.maxFileSize) {
        this.uploadError.emit({ 
          file, 
          error: `File size exceeds ${this.formatFileSize(this.maxFileSize)} limit` 
        });
        return false;
      }
      
      // Check file type
      if (this.acceptedTypes && !this.isAcceptedType(file)) {
        this.uploadError.emit({ 
          file, 
          error: `File type not accepted` 
        });
        return false;
      }
      
      return true;
    }).slice(0, this.maxFiles - this.files.length);
  }
  
  private isAcceptedType(file: File): boolean {
    const acceptedTypes = this.acceptedTypes.split(',').map(type => type.trim());
    return acceptedTypes.some(type => {
      if (type.startsWith('.')) {
        return file.name.toLowerCase().endsWith(type.toLowerCase());
      }
      return file.type.match(type.replace('*', '.*'));
    });
  }
  
  private addFiles(files: File[]) {
    this.files = [...this.files, ...files];
    
    // Initialize upload progress
    const newProgress = files.map(file => ({
      file,
      progress: 0,
      status: 'pending' as const
    }));
    
    this.uploadProgress = [...this.uploadProgress, ...newProgress];
    
    if (this.autoUpload) {
      this.startUpload(files);
    }
  }
  
  removeFile(file: File) {
    this.files = this.files.filter(f => f !== file);
    this.uploadProgress = this.uploadProgress.filter(p => p.file !== file);
  }
  
  private startUpload(files: File[]) {
    // Implementation would handle actual file upload
    // This is a simplified example
    files.forEach(file => {
      const progressItem = this.uploadProgress.find(p => p.file === file);
      if (progressItem) {
        progressItem.status = 'uploading';
        
        // Simulate upload progress
        const interval = setInterval(() => {
          progressItem.progress += 10;
          if (progressItem.progress >= 100) {
            progressItem.status = 'completed';
            clearInterval(interval);
          }
          this.uploadProgress.emit([...this.uploadProgress]);
        }, 200);
      }
    });
  }
  
  formatFileSize(bytes: number): string {
    if (bytes === 0) return '0 Bytes';
    
    const k = 1024;
    const sizes = ['Bytes', 'KB', 'MB', 'GB'];
    const i = Math.floor(Math.log(bytes) / Math.log(k));
    
    return parseFloat((bytes / Math.pow(k, i)).toFixed(2)) + ' ' + sizes[i];
  }
  
  trackFile(index: number, item: FileUploadProgress): string {
    return item.file.name + item.file.size;
  }
}
```

## Performance Optimizations

### Virtual Scrolling for Large Lists

```typescript
// virtual-scroll-drag.service.ts
import { Injectable } from '@angular/core';
import { fromEvent, Observable, combineLatest } from 'rxjs';
import { map, startWith, shareReplay } from 'rxjs/operators';

@Injectable({
  providedIn: 'root'
})
export class VirtualScrollDragService {
  
  createVirtualDragBehavior(
    container: HTMLElement,
    itemHeight: number,
    visibleItems: number
  ): Observable<any> {
    const scroll$ = fromEvent(container, 'scroll').pipe(
      map(() => container.scrollTop),
      startWith(0)
    );
    
    const resize$ = fromEvent(window, 'resize').pipe(
      map(() => ({
        width: container.clientWidth,
        height: container.clientHeight
      })),
      startWith({
        width: container.clientWidth,
        height: container.clientHeight
      })
    );
    
    return combineLatest([scroll$, resize$]).pipe(
      map(([scrollTop, dimensions]) => {
        const startIndex = Math.floor(scrollTop / itemHeight);
        const endIndex = Math.min(
          startIndex + visibleItems + 2, // Buffer items
          this.getTotalItems()
        );
        
        return {
          startIndex,
          endIndex,
          scrollTop,
          dimensions,
          translateY: startIndex * itemHeight
        };
      }),
      shareReplay(1)
    );
  }
  
  private getTotalItems(): number {
    // Would be implemented based on data source
    return 1000;
  }
}
```

## Testing Strategies

### Drag & Drop Testing Utilities

```typescript
// drag-drop-testing.utils.ts
import { ComponentFixture } from '@angular/core/testing';
import { DebugElement } from '@angular/core';
import { By } from '@angular/platform-browser';

export class DragDropTestingUtils {
  
  static simulateDragStart(
    fixture: ComponentFixture<any>,
    element: DebugElement,
    startPosition: { x: number; y: number }
  ) {
    const startEvent = new MouseEvent('mousedown', {
      clientX: startPosition.x,
      clientY: startPosition.y,
      bubbles: true
    });
    
    element.nativeElement.dispatchEvent(startEvent);
    fixture.detectChanges();
  }
  
  static simulateDragMove(
    fixture: ComponentFixture<any>,
    position: { x: number; y: number }
  ) {
    const moveEvent = new MouseEvent('mousemove', {
      clientX: position.x,
      clientY: position.y,
      bubbles: true
    });
    
    document.dispatchEvent(moveEvent);
    fixture.detectChanges();
  }
  
  static simulateDragEnd(fixture: ComponentFixture<any>) {
    const endEvent = new MouseEvent('mouseup', {
      bubbles: true
    });
    
    document.dispatchEvent(endEvent);
    fixture.detectChanges();
  }
  
  static simulateDrop(
    fixture: ComponentFixture<any>,
    dropZone: DebugElement,
    dragData: any,
    position: { x: number; y: number }
  ) {
    const dataTransfer = new DataTransfer();
    dataTransfer.setData('application/json', JSON.stringify(dragData));
    
    const dropEvent = new DragEvent('drop', {
      clientX: position.x,
      clientY: position.y,
      dataTransfer,
      bubbles: true
    });
    
    dropZone.nativeElement.dispatchEvent(dropEvent);
    fixture.detectChanges();
  }
  
  static findDraggableElements(fixture: ComponentFixture<any>): DebugElement[] {
    return fixture.debugElement.queryAll(By.css('[appDraggable]'));
  }
  
  static findDropZones(fixture: ComponentFixture<any>): DebugElement[] {
    return fixture.debugElement.queryAll(By.css('[appDropZone]'));
  }
}

// Example test
describe('KanbanBoardComponent', () => {
  let component: KanbanBoardComponent;
  let fixture: ComponentFixture<KanbanBoardComponent>;
  
  beforeEach(() => {
    // Setup component
  });
  
  it('should move item between columns', () => {
    const draggableItem = DragDropTestingUtils.findDraggableElements(fixture)[0];
    const targetDropZone = DragDropTestingUtils.findDropZones(fixture)[1];
    
    const dragData = { item: mockItem, columnId: 'column1', index: 0 };
    
    DragDropTestingUtils.simulateDragStart(fixture, draggableItem, { x: 100, y: 100 });
    DragDropTestingUtils.simulateDragMove(fixture, { x: 300, y: 100 });
    DragDropTestingUtils.simulateDrop(fixture, targetDropZone, dragData, { x: 300, y: 100 });
    
    expect(component.itemMoved.emit).toHaveBeenCalledWith({
      item: mockItem,
      fromColumn: 'column1',
      toColumn: 'column2',
      fromIndex: 0,
      toIndex: 0
    });
  });
});
```

## Best Practices & Common Patterns

### 1. **Memory Management**
- Always unsubscribe from drag observables
- Use takeUntil pattern for cleanup
- Debounce/throttle drag events for performance

### 2. **Accessibility**
- Provide keyboard navigation alternatives
- Include ARIA labels and roles
- Support screen readers with proper announcements

### 3. **Touch Support**
- Handle both mouse and touch events
- Prevent scroll during drag operations
- Support multi-touch gestures

### 4. **Performance**
- Use requestAnimationFrame for smooth animations
- Implement virtual scrolling for large lists
- Cache DOM queries and calculations

### 5. **User Experience**
- Provide visual feedback during drag operations
- Show drop zones and valid targets
- Implement undo/redo functionality

## Exercise: Build a File Manager

Create a file manager application that demonstrates:

1. **Draggable Files**: Files that can be dragged between folders
2. **Drop Zones**: Folders that accept specific file types
3. **Sortable Lists**: Files within folders can be reordered
4. **File Upload**: Drag files from desktop to upload
5. **Nested Folders**: Support for hierarchical folder structure

### Requirements:
- Implement proper validation and constraints
- Add animations and visual feedback
- Handle edge cases (invalid drops, file conflicts)
- Include comprehensive testing
- Optimize for performance with large file lists

## Summary

In this lesson, we've covered:

- ✅ Core drag & drop observable patterns
- ✅ Advanced draggable and drop zone directives
- ✅ Sortable list implementation
- ✅ Kanban board with cross-column moves
- ✅ File upload with drag & drop
- ✅ Performance optimizations
- ✅ Testing strategies
- ✅ Best practices and accessibility

These patterns provide the foundation for building sophisticated, interactive user interfaces with smooth drag and drop functionality using RxJS observables.

## Next Steps

In the next lesson, we'll explore **Real-World Case Studies & Examples**, examining how major applications implement complex reactive patterns and learning from their architectural decisions.
