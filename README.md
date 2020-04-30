# 基于列表类开发的列表组件

## 这只是部分代码，仅作参考

## 主要实现的功能

​	排序、分页、过滤、自定义操作列、列表中根据数据展示相应图标、跳转详情页

![image-20200429235125612](.\img\table.png)

​	

## 组件继承思想

​	封装一个列表基类，派生类继承该类，复用基类的功能。

##### Constructor 构造函数

​	如果派生类未声明构造函数，它将使用基类的构造函数。这意味着在基类构造函数注入的所有服务，子组件都能访问到。

##### 生命周期方法不继承

​	如果基类中包含生命周期钩子，如 ```ngOnInit```、```ngOnChanges``` 等，而在派生类没有定义相应的生命周期钩子，基类的生命周期钩子会被自动调用。如果需要在派生类触发```ngOnInit```，则需要在派生类上定义相应的生命周期钩子。

##### 继承的方法和属性基于可访问性级别

​	派生类不能访问私有方法和属性，仅继承公共方法和属性。

##### 模板是不能被继承

​	模板是不能被继承的 ，派生类需自定义模板，因此共享的 DOM 结构或行为需要单独处理。

##### 元数据和装饰器不继承

装饰器和元数据（```@Component```，```@Directive```，```@NgModule```等），这些元数据和装饰器不会被派生类继承，但是有一个例外，```@Input```和```@Output```装饰器会传递给派生类

##### 依赖注入

派生类必须通过调用super注入基类的参数

## 实现过程

application列表组件和基类```ResourceListBase```、```ResourceListWithStatuses``` 关系

![](.\img\列表数据分析体系.png)

application列表组件主要继承父类```ResourceListWithStatuses```，通过重写```ResourceListWithStatuses```的方法，自定义模板，来复用父类的功能属性。

#### 组件定义

##### 自定义模板

```ts
@Component({
  selector: 'sym-application-list',
  templateUrl: './application.component.html',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
```

application.component.html

```html
<div class="mat-elevation-z8">
  <div class="table-lf">
    <div class="header-top">
      <h4>Application</h4>
    </div>
  </div>

  <div class="table-content" content [hidden]="showZeroState()">
    <div class="table-loading" kdLoadingSpinner [isLoading]="isLoading"></div>
    <mat-table [dataSource]="getData()" matSort>
      <ng-container matColumnDef="statusicon">
        <mat-header-cell *matHeaderCellDef></mat-header-cell>
        <mat-cell *matCellDef="let element; let index=index">
          <mat-icon [ngClass]="getStatus(element).iconClass">
            <ng-container *ngIf="showHoverIcon(index, element); else showStatus">
              check_circle
            </ng-container>

            <ng-template #showStatus>
              {{getStatus(element).iconName}}
            </ng-template>
          </mat-icon>
        </mat-cell>
      </ng-container>
      <ng-container matColumnDef="name">
        <mat-header-cell *matHeaderCellDef mat-sort-header>Name</mat-header-cell>
        <mat-cell *matCellDef="let element">
          <a href="javascript:void(0);"
          (click)="goDetail(element)">
            {{element.objectMeta.name}}
          </a>
        </mat-cell>
      </ng-container>
  
      <ng-container matColumnDef="namespace">
        <mat-header-cell *matHeaderCellDef mat-sort-header>Namespace</mat-header-cell>
        <mat-cell *matCellDef="let element"> {{element.objectMeta.namespace}} </mat-cell>
      </ng-container>
  
      <ng-container matColumnDef="labels">
        <mat-header-cell *matHeaderCellDef>Labels</mat-header-cell>
        <mat-cell *matCellDef="let element">
          <sym-labels-group [value]='element.objectMeta.labels'></sym-labels-group>
        </mat-cell>
      </ng-container>
  
      <ng-container matColumnDef="replicas">
        <mat-header-cell *matHeaderCellDef mat-sort-header>Replicas</mat-header-cell>
        <mat-cell *matCellDef="let element">
          <a href="javascript:void(0);"
            *ngIf="element.status && element.status.replicas && element.status.availableReplicas ;else other">
            {{element.status.availableReplicas}} / {{element.status.replicas}}
          </a>
          <ng-template #other>
            <a href="javascript:void(0);" *ngIf="element.status.replicas && !element.status.availableReplicas">
              <span class='red'>0</span> / {{element.status.replicas}}
            </a>
            <a href="javascript:void(0);" *ngIf="!element.status.replicas && element.status.availableReplicas">
              {{element.status.availableReplicas}} / <span class='red'>0</span>
            </a>
            <a href="javascript:void(0);" class='red'
              *ngIf="!element.status.replicas && !element.status.availableReplicas">
              0 / 0
            </a>
          </ng-template>
        </mat-cell>
      </ng-container>
  
      <ng-container matColumnDef="creationTimestamp">
        <mat-header-cell *matHeaderCellDef mat-sort-header>Created Time </mat-header-cell>
        <mat-cell *matCellDef="let element"> {{element.status.creationTimestamp}} </mat-cell>
      </ng-container>
  
      <ng-container *ngFor="let col of getActionColumns()" [matColumnDef]="col.name">
        <mat-header-cell *matHeaderCellDef>Action</mat-header-cell>
        <mat-cell *matCellDef="let element"> 
          <kd-dynamic-cell [component]="col.component" [resource]="element"></kd-dynamic-cell>
        </mat-cell>
      </ng-container>

      <mat-header-row *matHeaderRowDef="getColumns()"></mat-header-row>
      <mat-row #matrow (mouseover)="onRowOver(index)" (mouseleave)="onRowLeave()" (click)="expand(index, row)"
        [ngClass]="{'kd-no-bottom-border': isRowExpanded(index), 'kd-clickable': hasErrors(row)}"
        *matRowDef="let row; columns: getColumns(); let index=index"></mat-row>
    </mat-table>

    <mat-paginator [length]="totalItems" [pageSize]="pageSize" [pageSizeOptions]="pageSizeOptions" showFirstLastButtons>
    </mat-paginator>
  </div>

  <div content [hidden]="!showZeroState()">
    <kd-list-zero-state></kd-list-zero-state>
  </div>
</div>
```

##### 继承父类

list.ts  基类定义

```ts
import { DataSource } from '@angular/cdk/collections';
import { HttpParams } from '@angular/common/http';
import {
  ChangeDetectorRef,
  ComponentFactoryResolver,
  EventEmitter,
  Inject,
  Input,
  OnDestroy,
  OnInit,
  AfterViewInit,
  Output,
  QueryList,
  Type,
  ViewChild,
  ViewChildren,
  ViewContainerRef,
} from '@angular/core';
import { MatPaginator, MatSort, MatTableDataSource } from '@angular/material';
import { Router } from '@angular/router';
import { Event as KdEvent, Resource, ResourceList } from '@api/symui';
import {
  ActionColumn,
  ActionColumnDef,
  ColumnWhenCallback,
  ColumnWhenCondition,
  OnListChangeEvent,
} from '@api/symui';
import { Subject } from 'rxjs';
import { Observable, ObservableInput } from 'rxjs/Observable';
import { merge } from 'rxjs/observable/merge';
import { startWith, switchMap, takeUntil } from 'rxjs/operators';

// import { SEARCH_QUERY_STATE_PARAM } from '../params/params';
import { KdStateService } from '@service/global/state.service';
import { LocaltionService } from '@service/global/localtion.service';
import { CardListFilterComponent } from '../components/list/filter/filter.component';
import { NotificationsService } from '@service/global/notifications.service';
import { GlobalServicesModule } from '@service/global/global.module';
import { RowDetailComponent } from '../components/list/rowdetail/rowdetail.component';
import { ConfigService } from '@service/global/config.service';

export abstract class ResourceListBase<T extends ResourceList, R extends Resource>
  implements OnInit, OnDestroy, AfterViewInit {
  // Base properties
  private readonly actionColumns_: Array<ActionColumnDef<ActionColumn>> = [];
  public readonly data_ = new MatTableDataSource<R>();
  private listUpdates_ = new Subject();
  private unsubscribe_ = new Subject<void>();
  private loaded_ = false;
  private readonly dynamicColumns_: ColumnWhenCondition[] = [];
  // private paramsService_: ParamsService;
  private router_: Router;
  protected readonly kdState_: KdStateService;
  protected readonly settingsService_: ConfigService;
  // protected readonly namespaceService_: NamespaceService;
  protected readonly localtionService_: LocaltionService;

  isLoading = false;
  totalItems = 0;

  get itemsPerPage(): number {
    return this.settingsService_.getItemsPerPage();
  }

  @Output('onchange') onChange: EventEmitter<OnListChangeEvent> = new EventEmitter();

  @Input() groupId: string;
  @Input() hideable = false;
  @Input() id: string;

  // Data select properties
  @ViewChild(MatSort, { static: false }) private readonly matSort_: MatSort;
  @ViewChild(MatPaginator, { static: false }) private readonly matPaginator_: MatPaginator;
  @ViewChild(CardListFilterComponent, { static: false })
  private readonly cardFilter_: CardListFilterComponent;

  protected constructor(
    private readonly stateName_: string,
    private readonly notifications_: NotificationsService,
    private readonly cdr_: ChangeDetectorRef,
  ) {
    // this.settingsService_ = GlobalServicesModule.injector.get(GlobalSettingsService);
    this.kdState_ = GlobalServicesModule.injector.get(KdStateService);
    // this.namespaceService_ = GlobalServicesModule.injector.get(NamespaceService);
    // this.paramsService_ = GlobalServicesModule.injector.get(ParamsService);
    this.router_ = GlobalServicesModule.injector.get(Router);
    this.localtionService_ = GlobalServicesModule.injector.get(LocaltionService);
    this.settingsService_ = GlobalServicesModule.injector.get(ConfigService);
  }

  ngOnInit(): void {
    if (!this.id) {
      throw Error('ID is a required attribute of list component.');
    }

    this.getList();

    // if (this.matPaginator_ === undefined) {
    //   throw Error('MatPaginator has to be defined on a table.');
    // }

    // this.namespaceService_.onNamespaceChangeEvent.subscribe(() => {
    //   this.isLoading = true;
    //   this.listUpdates_.next();
    // });

    // this.paramsService_.onParamChange.subscribe(() => {
    //   this.isLoading = true;
    //   this.listUpdates_.next();
    // });
  }

  ngAfterViewInit(): void {
    this.data_.sort = this.matSort_;
    this.data_.paginator = this.matPaginator_;

    if (this.cardFilter_) {
      // 表格搜索
      this.cardFilter_.filterEvent.subscribe(() => {
        const filterValue = this.cardFilter_.query;
        this.data_.filter = filterValue.trim().toLowerCase();
      });
    }
  }

  ngOnDestroy(): void {
    this.unsubscribe_.next();
    this.unsubscribe_.complete();
  }

  getDetailsHref(resourceName: string, namespace?: string): string {
    return this.stateName_ ? this.kdState_.href(this.stateName_, resourceName,
      this.localtionService_.current().project + '-' + namespace) : '';
  }

  getParams() {
    return {
      cluster: this.localtionService_.current().cluster
    };
  }

  getData(): DataSource<R> {
    return this.data_;
  }

  getList() {
    this.getObservableWithDataSelect_()
      .pipe(startWith({}))
      .pipe(
        switchMap(() => {
          this.isLoading = true;
          if (this.cdr_) {
            this.cdr_.markForCheck();
            this.cdr_.detectChanges();
          }
          // 暂时去掉排序分页参数
          // return this.getResourceObservable(this.getDataSelectParams_());
          return this.getResourceObservable();
        }),
      )
      .pipe(takeUntil(this.unsubscribe_))
      .subscribe((data: T) => {
        this.notifications_.pushErrors(data.errors);
        this.data_.data = this.map(data);
        this.totalItems = data.listMeta.totalItems;
        this.isLoading = false;
        this.loaded_ = true;

        this.onListChange_(data);

        if (this.cdr_) {
          this.cdr_.detectChanges();
        }
      });
  }

  showZeroState(): boolean {
    return this.totalItems === 0 && !this.isLoading;
  }

  isHidden(): boolean {
    return this.hideable && !this.filtered_() && this.showZeroState();
  }

  getColumns(): string[] {
    const displayColumns = this.getDisplayColumns();
    const actionColumns = this.actionColumns_.map(col => col.name);

    for (const condition of this.dynamicColumns_) {
      if (condition.whenCallback()) {
        const afterColIdx = displayColumns.indexOf(condition.afterCol);
        displayColumns.splice(afterColIdx + 1, 0, condition.col);
      }
    }

    return displayColumns.concat(...actionColumns);
  }

  getActionColumns(): Array<ActionColumnDef<ActionColumn>> {
    return this.actionColumns_;
  }

  shouldShowColumn(dynamicColName: string): boolean {
    const col = this.dynamicColumns_.find(condition => {
      return condition.col === dynamicColName;
    });
    if (col !== undefined) {
      return col.whenCallback();
    }

    return false;
  }

  protected registerActionColumn<C extends ActionColumn>(name: string, component: Type<C>): void {
    this.actionColumns_.push({
      name: `action-${name}`,
      component,
    } as ActionColumnDef<ActionColumn>);
  }

  protected registerDynamicColumn(
    col: string,
    afterCol: string,
    whenCallback: ColumnWhenCallback,
  ): void {
    this.dynamicColumns_.push({
      col,
      afterCol,
      whenCallback,
    } as ColumnWhenCondition);
  }

  private getObservableWithDataSelect_<E>(): Observable<E> {
    // const obsInput = [this.matPaginator_.page] as Array<ObservableInput<E>>;
    const obsInput = [] as Array<ObservableInput<E>>;

    // 暂时去掉排序请求
    // if (this.matSort_) {
    //   this.matSort_.sortChange.subscribe(() => (this.matPaginator_.pageIndex = 0));
    //   obsInput.push(this.matSort_.sortChange);
    // }

    // 暂时去掉字段过滤
    // if (this.cardFilter_) {
      // this.cardFilter_.filterEvent.subscribe(() => (this.matPaginator_.pageIndex = 0));
      // obsInput.push(this.cardFilter_.filterEvent);
    // }

    return merge(...obsInput, this.listUpdates_ as Subject<E>);
  }

  private getDataSelectParams_(): HttpParams {
    let params = this.paginate_();

    if (this.matSort_) {
      params = this.sort_(params);
    }

    if (this.cardFilter_) {
      params = this.filter_(params);
    }

    return this.search_(params);
  }

  private sort_(params?: HttpParams): HttpParams {
    let result = new HttpParams();
    if (params) {
      result = params;
    }

    return result.set('sortBy', this.getSortBy_());
  }

  private paginate_(params?: HttpParams): HttpParams {
    let result = new HttpParams();
    if (params) {
      result = params;
    }

    return result
      .set('itemsPerPage', `${this.itemsPerPage}`)
      .set('page', `${this.matPaginator_.pageIndex + 1}`);
  }

  private filter_(params?: HttpParams): HttpParams {
    let result = new HttpParams();
    if (params) {
      result = params;
    }

    const filterByQuery = this.cardFilter_.query ? `name,${this.cardFilter_.query}` : '';
    if (filterByQuery) {
      return result.set('filterBy', filterByQuery);
    }

    return result;
  }

  private search_(params?: HttpParams): HttpParams {
    let result = new HttpParams();
    if (params) {
      result = params;
    }

    const filterByQuery = result.get('filterBy') || '';
    if (this.router_.routerState.snapshot.url.startsWith('/search')) {
      // const query = this.paramsService_.getQueryParam(SEARCH_QUERY_STATE_PARAM);
      // if (query) {
      //   if (filterByQuery) {
      //     filterByQuery += ',';
      //   }
      //   filterByQuery += `name,${query}`;
      // }
    }

    if (filterByQuery) {
      return result.set('filterBy', filterByQuery);
    }
    return result;
  }

  private filtered_(): boolean {
    return !!this.filter_().get('filterBy');
  }

  private getSortBy_(): string {
    // Default values.
    let ascending = true;
    let active = 'age';

    if (this.matSort_.direction) {
      ascending = this.matSort_.direction === 'asc';
    }

    if (this.matSort_.active) {
      active = this.matSort_.active;
    }

    if (active === 'age') {
      ascending = !ascending;
    }

    return `${ascending ? 'a' : 'd'},${this.mapToBackendValue_(active)}`;
  }

  private mapToBackendValue_(sortByColumnName: string): string {
    return sortByColumnName === 'age' ? 'creationTimestamp' : sortByColumnName;
  }

  private onListChange_(data: T): void {
    const emitValue = {
      id: this.id,
      groupId: this.groupId,
      items: this.totalItems,
      filtered: false,
      resourceList: data,
    } as OnListChangeEvent;

    if (this.cardFilter_) {
      emitValue.filtered = this.filtered_();
    }

    this.onChange.emit(emitValue);
  }

  protected abstract getDisplayColumns(): string[];

  abstract getResourceObservable(params?: HttpParams): Observable<T>;

  abstract map(value: T): R[];
}

export abstract class ResourceListWithStatuses<
  T extends ResourceList,
  R extends Resource
  > extends ResourceListBase<T, R> {
  private readonly bindings_: { [hash: number]: StateBinding<R> } = {};
  @ViewChildren('matrow', { read: ViewContainerRef })
  private readonly containers_: QueryList<ViewContainerRef>;
  private lastHash_: number;
  private readonly unknownStatus: StatusIcon = {
    iconName: 'help',
    iconClass: { '': true },
  };

  protected icon = IconName;

  expandedRow: number = undefined;
  hoveredRow: number = undefined;

  protected constructor(
    stateName: string,
    private readonly notifications: NotificationsService,
    cdr: ChangeDetectorRef,
    private readonly resolver_?: ComponentFactoryResolver,
  ) {
    super(stateName, notifications, cdr);

    this.onChange.subscribe(this.clearExpandedRows_.bind(this));
  }

  expand(index: number, resource: R): void {
    if (!this.hasErrors(resource)) {
      return;
    }

    if (this.expandedRow !== undefined) {
      this.containers_.toArray()[this.expandedRow].clear();
    }

    if (this.expandedRow === index) {
      this.expandedRow = undefined;
      return;
    }

    const container = this.containers_.toArray()[index];
    const factory = this.resolver_.resolveComponentFactory(RowDetailComponent);
    const component = container.createComponent(factory);

    component.instance.events = this.getEvents(resource);
    this.expandedRow = index;
  }


  compare(a: number | string, b: number | string, isAsc: boolean) {
    return (a < b ? -1 : 1) * (isAsc ? 1 : -1);
  }

  getStatus(resource: R): StatusIcon {
    if (this.lastHash_) {
      const stateBinding = this.bindings_[this.lastHash_];
      if (stateBinding.callbackFunction(resource)) {
        return this.getStatusObject_(stateBinding);
      }
    }

    // map() is needed here to cast hash from string to number. Without it compiler will not
    // recognize stateBinding type.
    for (const hash of Object.keys(this.bindings_).map((hashStr): number => Number(hashStr))) {
      const stateBinding = this.bindings_[hash];
      if (stateBinding.callbackFunction(resource)) {
        this.lastHash_ = Number(hash);
        return this.getStatusObject_(stateBinding);
      }
    }

    return this.unknownStatus;
  }

  isRowExpanded(index: number): boolean {
    return this.expandedRow === index;
  }

  isRowHovered(index: number): boolean {
    return this.hoveredRow === index;
  }

  onRowOver(rowIdx: number): void {
    this.hoveredRow = rowIdx;
  }

  onRowLeave(): void {
    this.hoveredRow = undefined;
  }

  showHoverIcon(index: number, resource: R): boolean {
    return this.isRowHovered(index) && this.hasErrors(resource) && !this.isRowExpanded(index);
  }

  protected getEvents(_resource: R): KdEvent[] {
    return [];
  }

  protected hasErrors(_resource: R): boolean {
    return false;
  }

  protected registerBinding(
    iconName: IconName,
    iconClass: string,
    callbackFunction: StatusCheckCallback<R>,
  ): void {
    const icon = new Icon(String(iconName), iconClass);
    this.bindings_[icon.hash()] = { icon, callbackFunction };
  }

  private clearExpandedRows_(): void {
    const containers = this.containers_.toArray();
    for (let i = 0; i < containers.length; i++) {
      containers[i].clear();
      this.expandedRow = undefined;
    }
  }

  private getStatusObject_(stateBinding: StateBinding<R>): StatusIcon {
    return {
      iconName: stateBinding.icon.name,
      iconClass: { [stateBinding.icon.cssClass]: true },
    };
  }
}

interface StatusIcon {
  iconName: string;
  iconClass: { [className: string]: boolean };
}

enum IconName {
  error = 'error',
  timelapse = 'timelapse',
  checkCircle = 'check_circle',
  help = 'help',
  warning = 'warning',
  none = '',
}

class Icon {
  name: string;
  cssClass: string;

  constructor(name: string, cssClass: string) {
    this.name = name;
    this.cssClass = cssClass;
  }

  /**
   * Implementation of djb2 hash function:
   * http://www.cse.yorku.ca/~oz/hash.html
   */
  hash(): number {
    const value = `${this.name}#${this.cssClass}`;
    return value
      .split('')
      .map(str => {
        return str.charCodeAt(0);
      })
      .reduce((prev, curr) => {
        return (prev << 5) + prev + curr;
      }, 5381);
  }
}

type StatusCheckCallback<T> = (resource: T) => boolean;

interface StateBinding<T> {
  icon: Icon;
  callbackFunction: StatusCheckCallback<T>;
}
```

application.component.ts

```ts
export class ApplicationListComponent extends ResourceListWithStatuses<ApplicationList, Application> {
  @Input() endpoint = EndpointManager.resource(Resource.application, true).list();
  @Input() showMetrics = false;
  @Input() initialized: boolean;
  @Input() deployment: string;

  cumulativeMetrics: Metric[];
  pageSize = 5;
  pageSizeOptions: number[] = [5, 10, 25, 100];
  namespace: string;
  nsList: any;

  constructor(
    private router: Router,
    private readonly application_: NamespacedResourceService<ApplicationList>,
    resolver: ComponentFactoryResolver,
    notifications: NotificationsService,
    cdr: ChangeDetectorRef,
  ) {
    super('application', notifications, cdr, resolver);
    this.id = ListIdentifier.application;
    this.groupId = ListGroupIdentifier.workloads;
    // Register status icon handlers
    this.registerBinding(this.icon.checkCircle, 'kd-success', this.isInSuccessState);
    this.registerBinding(this.icon.error, 'kd-error', this.isInErrorState);

    // Register action columns.
    this.registerActionColumn<MenuComponent>('menu', MenuComponent);
  }
  .....
  }
```

##### 重写父类方法

通过继承基类```ResourceListWithStatuses```，可以复用其基本的功能属性，同时可以定义属性和方法扩展列表组件的功能，实现这个列表组件主要的是如果**调用组件方法发起请求，获取到数据**

- 重写```getResourceObservable``` 方法，该方法主要是返回一个Observable，进行异步编程。

  在这个方法里，可以请求多个api，使用Observable.forkJoin，可以合并多个Observable，返回一个Observable。

- 结合```sym-select``` 组件搜索

  ![image-20200430175615525](.\img\symSelect.png)

  ```sym-select```组件的namespace下拉列表的数据nsList是异步获取的，如果是选择是的namespace是ALL，那么列表组件要获取到nsList，然后对每个nsList里的namespace进行全部请求。如果选择的是某个namesapce，那么就传入namespace的值，进行单个请求

  **这里也是比较疑惑的地方**，之前想通过```@Input```传入属性的方式，来把搜索组件的数据传递给applicaiton列表组件```sym-application-list```

  ![](.\img\input.png)

  但是尝试之后，发现在页面初始化的时候，通过```@Input```的属性传入的```nsList```为空的。如果传入的属性是同步获取的，则可以传递到application列表组件中

  最后获取搜索参数的方式，采取**服务依赖注入**的方式来获取。

  ![](.\img\subscribe.png)

- 通过重写map 方法，在自定义表格的数据，返回所需表格的数据。

```ts
ngOnInit() {
    this.data_.sortingDataAccessor = (item, property) => {
      switch (property) {
        case 'name': return item.objectMeta.name;
        case 'creationTimestamp': return item.status.creationTimestamp;
        case 'namespace': return item.objectMeta.namespace;
        default: return item[property];
      }
    };
  }

  getResourceObservable(params?: HttpParams): Observable<ApplicationList> {
    const res = this.localtionService_.onNamespaceUpdate.subscribe(() => {
      this.namespace = this.localtionService_.current().namespace;
      this.nsList = this.localtionService_.current().namespaceList;
    });
    const data: any = {
      items: [],
      listMeta: {
        totalItems: 0
      }
    };
    if (this.namespace === 'ALL') {
      const list = this.nsList.slice(1);
      const observableList = list.map((ns: any) => {
        return this.application_.get(this.endpoint, undefined, ns.name);
      });
      if (observableList && observableList.length === 0) {
        return new Observable((observer) => {
          observer.next(data);
        });
      }
      return new Observable((observer) => {
        Observable.forkJoin(observableList).subscribe((res: any) => {
          res.map((item: any, index: number) => {
            item.items = item.items.map((v: any) => {
              v.objectMeta.namespace = list[index].name;
              return v;
            });
            data.items = data.items.concat(item.items);
            data.listMeta.totalItems += item.listMeta.totalItems;
          });
          observer.next(data);
        });
      });
    } else if (this.namespace !== undefined) {
      return this.application_.get(this.endpoint, undefined, this.namespace, params);
    }
    return new Observable((observer) => {
      observer.next(data);
    });
  }

  map(applicationList: ApplicationList): Application[] {
    if (this.namespace !== 'ALL') {
      applicationList.items = applicationList.items.map((v: any) => {
        v.objectMeta.namespace = this.namespace;
        return v;
      });
    }
    console.log('applicationList', applicationList);
    return applicationList.items;
  }

  isInErrorState(resource: Application): boolean {
    return resource.status ? (resource.status.replicas && resource.status.replicas !== resource.status.availableReplicas ?
       true : false) : false;
  }

  isInSuccessState(resource: Application): boolean {
    return resource.status ? (resource.status.replicas && resource.status.replicas === resource.status.availableReplicas ?
       true : false) : false;
  }

  protected getDisplayColumns(): string[] {
    return ['statusicon', 'name', 'namespace', 'labels', 'replicas', 'creationTimestamp'];
  }
```

#### 组件使用

```html
<sym-application-list #applicationList></sym-application-list>
```

初始化时，调用getList请求方法

```ts
this.applicationList.getList();
```

## 公共组件的巧妙之处

##### actionColumn 操作列可以自定义

在基类中已有声明注册操作列的方法，在application列表组件继承该方法，在构造函数中，注册操作列，```MenuComponent```是引入的组件

![image-20200430184244832](.\img\action.png)

```
this.registerActionColumn<MenuComponent>('menu', MenuComponent);
```

##### 过滤功能

![image-20200430184348308](.\img\filter.png)

这里的Pods列表组件也是继承了```ResourceListWithStatuses```

继承的组件都会继承过滤功能，因为继承的组件的模板是自定义的，可以在模板上引入过滤的组件

```html
<kd-card-list-filter></kd-card-list-filter>
```

基类中以判断是否有过滤组件，并订阅过滤组件的方法触发列表过滤

```ts
ngAfterViewInit(): void {
    this.data_.sort = this.matSort_;
    this.data_.paginator = this.matPaginator_;

    if (this.cardFilter_) {
      // 表格搜索
      this.cardFilter_.filterEvent.subscribe(() => {
        const filterValue = this.cardFilter_.query;
        this.data_.filter = filterValue.trim().toLowerCase();
      });
    }
  }
```

