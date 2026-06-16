# 🅰️ Angular Complete Q&A — Interview Discussion
> All questions and answers from our discussion
> Full-stack interview context | Angular 8 to Angular 17
> Format: Question → What I Said → What I Should Have Said → One-Liner

---

## 📌 HOW TO USE THIS FILE
```
1. Cover the answer
2. Try to answer from memory out loud
3. Uncover and compare
4. Mark: ❌ Wrong | 🟡 Partial | ✅ Nailed it
5. Revisit ❌ and 🟡 daily until all ✅
```

---

## 🔷 SECTION 1 — CORE CONCEPTS

### Q1: What is Angular and how does it differ from React?

**✅ Complete Answer:**

> *"Angular is a complete opinionated framework from Google — it comes with everything built in: routing, HTTP client, forms, dependency injection, and testing utilities. You don't need to choose third-party libraries for these. React is a UI library from Meta — it only handles the view layer. You need to choose your own router (React Router), state management (Redux/Zustand), HTTP client (axios), and so on.*
>
> *Angular uses TypeScript by default, component-based architecture, and two-way data binding with NgModel. React uses JSX and one-way data flow with props and state. Angular has a steeper learning curve but gives you more structure — great for large enterprise teams where consistency matters. React is more flexible but requires more architectural decisions upfront.*
>
> *For Barclays or any large financial institution, Angular's opinionated structure actually works in your favour — every team builds features the same way."*

**One-liner:**
> *"Angular = complete framework with everything included. React = UI library, you choose the rest. Angular for enterprise consistency, React for flexibility."*

---

### Q2: What are the 4 types of data binding in Angular?

**✅ Complete Answer:**

```typescript
// 1. INTERPOLATION — component → template (display values)
<h1>{{ employee.name }}</h1>
<p>{{ 'Hello ' + employee.name }}</p>
<p>{{ employee.salary | currency }}</p>
// Read-only, one-way from component to view

// 2. PROPERTY BINDING — component → template (set DOM properties)
<img [src]="employee.photoUrl">
<button [disabled]="isLoading">Save</button>
<input [value]="employee.name">
// Square brackets = binding to DOM property

// 3. EVENT BINDING — template → component (listen to events)
<button (click)="saveEmployee()">Save</button>
<input (keyup)="onKeyUp($event)">
<form (ngSubmit)="onSubmit()">
// Round brackets = listening to events

// 4. TWO-WAY BINDING — both directions simultaneously
<input [(ngModel)]="employee.name">
// Banana in a box syntax [()]
// Requires FormsModule imported in module
// Equivalent to: [value]="name" (input)="name = $event.target.value"
```

**One-liner:**
> *"Interpolation {{ }} displays values. Property binding [ ] sets DOM properties. Event binding ( ) listens to user actions. Two-way binding [(ngModel)] does both — requires FormsModule."*

---

### Q3: What are the lifecycle hooks and in what order do they run?

**✅ Complete Answer:**

```typescript
export class EmployeeComponent implements OnInit, OnChanges,
                                          AfterViewInit, OnDestroy {

  @Input() employeeId: number;

  // 1. ngOnChanges — FIRST, before OnInit
  // Called when @Input() values change
  // Called every time input changes (not just once)
  ngOnChanges(changes: SimpleChanges): void {
    if (changes['employeeId']) {
      console.log('ID changed:', changes['employeeId'].currentValue);
    }
  }

  // 2. ngOnInit — called ONCE after first ngOnChanges
  // Best place for: API calls, initialisation
  ngOnInit(): void {
    this.loadEmployee();
  }

  // 3. ngAfterViewInit — after view and child views initialised
  // Best place for: DOM manipulation, @ViewChild access
  ngAfterViewInit(): void {
    // @ViewChild elements are NOW available
  }

  // 4. ngOnDestroy — just before component destroyed
  // Best place for: unsubscribe, cleanup, clear timers
  ngOnDestroy(): void {
    this.subscription.unsubscribe();
  }
}
```

**Order to memorise:**
```
OnChanges → OnInit → AfterContentInit → AfterContentChecked
→ AfterViewInit → AfterViewChecked → OnDestroy

Key 4: OnChanges → OnInit → AfterViewInit → OnDestroy
```

**One-liner:**
> *"OnChanges fires first when inputs change. OnInit fires once for initialisation. AfterViewInit when DOM is ready. OnDestroy for cleanup. Never put API calls in constructor — use OnInit."*

---

### Q4: What is the difference between constructor and ngOnInit?

**✅ Complete Answer:**

> *"Constructor is a TypeScript/JavaScript concept — it runs when the class is instantiated by Angular's dependency injection system. At this point, Angular has NOT yet set @Input() values and the view is NOT yet created. You should ONLY use constructor for dependency injection — injecting services.*
>
> *ngOnInit is Angular's lifecycle hook — it runs after Angular has initialised the component, set all @Input() bindings, and prepared the component for use. This is where you put initialisation logic: API calls, subscriptions, computing values from inputs.*
>
> *Rule: Constructor = inject dependencies only. ngOnInit = everything else."*

```typescript
export class EmployeeComponent implements OnInit {
  employee: Employee;

  // Constructor: ONLY for DI — inputs NOT yet set
  constructor(
    private employeeService: EmployeeService,
    private router: Router
  ) {
    // DON'T do this — employeeId @Input() not set yet
    // this.loadEmployee(this.employeeId); // WRONG
  }

  // ngOnInit: inputs ARE set, safe to use them
  ngOnInit(): void {
    this.loadEmployee(this.employeeId); // CORRECT
  }
}
```

---

### Q5: What are directives? What are the 3 types?

**✅ Complete Answer:**

```typescript
// TYPE 1: STRUCTURAL DIRECTIVES — change DOM structure (add/remove elements)
// Always start with *

<div *ngIf="isLoggedIn">Welcome!</div>
<div *ngIf="isLoggedIn; else loginBlock">Welcome!</div>
<ng-template #loginBlock>Please login</ng-template>

<li *ngFor="let emp of employees; let i = index; trackBy: trackById">
  {{ i + 1 }}. {{ emp.name }}
</li>

<div [ngSwitch]="userRole">
  <p *ngSwitchCase="'admin'">Admin Panel</p>
  <p *ngSwitchCase="'user'">Dashboard</p>
  <p *ngSwitchDefault>Guest</p>
</div>

// TYPE 2: ATTRIBUTE DIRECTIVES — change appearance/behaviour
<div [ngClass]="{ 'active': isActive, 'error': hasError }">
<div [ngStyle]="{ 'color': isActive ? 'green' : 'red', 'font-size': '14px' }">

// TYPE 3: CUSTOM DIRECTIVES — your own behaviour
@Directive({ selector: '[appHighlight]' })
export class HighlightDirective {
  constructor(private el: ElementRef) {}

  @HostListener('mouseenter') onMouseEnter() {
    this.el.nativeElement.style.backgroundColor = 'yellow';
  }
  @HostListener('mouseleave') onMouseLeave() {
    this.el.nativeElement.style.backgroundColor = null;
  }
}
// Usage: <p appHighlight>Hover me</p>
```

**One-liner:**
> *"Structural directives (*ngIf, *ngFor) add or remove DOM elements. Attribute directives ([ngClass], [ngStyle]) change appearance. Custom directives add your own behaviour to any element."*

---

## 🔷 SECTION 2 — PARENT-CHILD COMMUNICATION

### Q6: How does Parent → Child communication work in Angular?

**✅ Complete Answer:**

```typescript
// CHILD COMPONENT — receives data via @Input()
@Component({
  selector: 'app-employee-card',
  template: `
    <h3>{{ employee.name }}</h3>
    <p *ngIf="showSalary">{{ employee.salary | currency }}</p>
  `
})
export class EmployeeCardComponent implements OnChanges {
  @Input() employee: Employee;           // required input
  @Input() showSalary: boolean = false;  // optional with default

  ngOnChanges(changes: SimpleChanges): void {
    // Called whenever @Input() value changes
    if (changes['employee']) {
      console.log('Employee updated:', changes['employee'].currentValue);
    }
  }
}

// PARENT TEMPLATE — passes data with property binding
<app-employee-card
  [employee]="selectedEmployee"
  [showSalary]="isManager">
</app-employee-card>
```

**One-liner:**
> *"@Input() decorator marks a property as receivable from parent. Parent uses property binding [property]='value' to pass data down."*

---

### Q7: How does Child → Parent communication work?

**✅ Complete Answer:**

```typescript
// CHILD — emits events via @Output() + EventEmitter
@Component({
  selector: 'app-employee-card',
  template: `
    <button (click)="onSelect()">Select Employee</button>
    <button (click)="onDelete()">Delete</button>
  `
})
export class EmployeeCardComponent {
  @Input() employee: Employee;
  @Output() employeeSelected = new EventEmitter<Employee>();
  @Output() employeeDeleted = new EventEmitter<number>();

  onSelect(): void {
    this.employeeSelected.emit(this.employee); // send entire object
  }

  onDelete(): void {
    this.employeeDeleted.emit(this.employee.id); // send just the ID
  }
}

// PARENT TEMPLATE — listens with event binding
<app-employee-card
  [employee]="emp"
  (employeeSelected)="handleSelected($event)"
  (employeeDeleted)="handleDelete($event)">
</app-employee-card>

// PARENT COMPONENT
handleSelected(employee: Employee): void {
  this.selectedEmployee = employee;
  console.log('Selected:', employee.name);
}

handleDelete(id: number): void {
  this.employees = this.employees.filter(e => e.id !== id);
}
```

**One-liner:**
> *"@Output() with EventEmitter lets child emit events to parent. Parent listens with (eventName)='handler($event)'. The $event contains whatever was emitted."*

---

### Q8: How do you access a child component directly from parent? (@ViewChild)

**✅ Complete Answer:**

```typescript
// CHILD COMPONENT
@Component({ selector: 'app-timer' })
export class TimerComponent {
  isRunning = false;
  elapsedSeconds = 0;

  start(): void { this.isRunning = true; }
  stop(): void  { this.isRunning = false; }
  reset(): void { this.isRunning = false; this.elapsedSeconds = 0; }
}

// PARENT — access child via @ViewChild
@Component({
  template: `
    <app-timer #timerRef></app-timer>
    <button (click)="startTimer()">Start</button>
    <button (click)="stopTimer()">Stop</button>
    <button (click)="resetTimer()">Reset</button>
  `
})
export class ParentComponent implements AfterViewInit {
  @ViewChild('timerRef') timer: TimerComponent;
  // IMPORTANT: only available AFTER ngAfterViewInit
  // NOT available in ngOnInit!

  ngAfterViewInit(): void {
    console.log('Timer ready:', this.timer);
  }

  startTimer(): void { this.timer.start(); }
  stopTimer(): void  { this.timer.stop(); }
  resetTimer(): void { this.timer.reset(); }
}
```

**One-liner:**
> *"@ViewChild gives parent direct access to child component's methods and properties. Available only in ngAfterViewInit, not ngOnInit."*

---

### Q9: How do sibling components communicate?

**✅ Complete Answer:**

```typescript
// SHARED SERVICE — the communication bridge between siblings
@Injectable({ providedIn: 'root' })
export class EmployeeSharedService {
  // BehaviorSubject holds current value + emits to new subscribers
  private selectedEmployeeSubject = new BehaviorSubject<Employee>(null);
  selectedEmployee$ = this.selectedEmployeeSubject.asObservable();

  selectEmployee(employee: Employee): void {
    this.selectedEmployeeSubject.next(employee);
  }

  clearSelection(): void {
    this.selectedEmployeeSubject.next(null);
  }
}

// SIBLING A — SENDER (employee list)
@Component({ selector: 'app-employee-list' })
export class EmployeeListComponent {
  constructor(private sharedService: EmployeeSharedService) {}

  onEmployeeClick(employee: Employee): void {
    this.sharedService.selectEmployee(employee); // broadcast
  }
}

// SIBLING B — RECEIVER (employee detail)
@Component({ selector: 'app-employee-detail' })
export class EmployeeDetailComponent implements OnInit, OnDestroy {
  employee: Employee;
  private destroy$ = new Subject<void>();

  constructor(private sharedService: EmployeeSharedService) {}

  ngOnInit(): void {
    this.sharedService.selectedEmployee$
      .pipe(takeUntil(this.destroy$))
      .subscribe(emp => {
        if (emp) this.employee = emp;
      });
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

**One-liner:**
> *"Siblings communicate via a shared service using BehaviorSubject. Sender calls service method to update. Receiver subscribes to Observable. Always unsubscribe in ngOnDestroy."*

---

### Q10: What is ng-content and content projection?

**✅ Complete Answer:**

```typescript
// ng-content = like React's children prop
// Allows parent to inject HTML into a reusable component

// REUSABLE CARD COMPONENT
@Component({
  selector: 'app-card',
  template: `
    <div class="card">
      <div class="card-header">
        <ng-content select="[card-title]"></ng-content>
      </div>
      <div class="card-body">
        <ng-content select="[card-body]"></ng-content>
      </div>
      <div class="card-footer">
        <ng-content select="[card-footer]"></ng-content>
      </div>
    </div>
  `
})
export class CardComponent {}

// USAGE — parent controls the content
<app-card>
  <h2 card-title>Employee Details</h2>
  <div card-body>
    <p>{{ employee.name }}</p>
    <p>{{ employee.department }}</p>
  </div>
  <button card-footer (click)="save()">Save Changes</button>
</app-card>

// SIMPLE SINGLE SLOT — no select needed
@Component({
  selector: 'app-modal',
  template: `
    <div class="modal-overlay">
      <div class="modal-box">
        <ng-content></ng-content>
      </div>
    </div>
  `
})
export class ModalComponent {}

<app-modal>
  <h2>Are you sure?</h2>
  <p>This action cannot be undone.</p>
  <button>Confirm</button>
</app-modal>
```

**One-liner:**
> *"ng-content enables content projection — like React children. Parent controls what goes inside a reusable component. Named slots with select attribute allow multiple injection points."*

---

## 🔷 SECTION 3 — SERVICES & DEPENDENCY INJECTION

### Q11: What is Dependency Injection and why does Angular use it?

**✅ Complete Answer:**

> *"Dependency Injection means Angular creates and manages service instances for you — you just declare what you need in the constructor, Angular provides it. Without DI, each component would create its own service instance — leading to multiple database connections, multiple copies of state, and untestable code. With DI, Angular creates ONE instance of a service (singleton) and shares it everywhere it's needed.*
>
> *The huge benefit for testing: in unit tests you can provide a MOCK service instead of the real one, without changing any component code. The component doesn't care whether it's getting the real service or a mock — it just uses whatever Angular gives it."*

```typescript
// Service — marked as injectable
@Injectable({
  providedIn: 'root'  // singleton for entire app
})
export class EmployeeService {
  constructor(private http: HttpClient) {}

  getEmployees(): Observable<Employee[]> {
    return this.http.get<Employee[]>('/api/employees');
  }
}

// Component — declares dependency, Angular provides it
@Component({...})
export class EmployeeListComponent implements OnInit {
  employees: Employee[] = [];

  // Angular injects EmployeeService automatically
  constructor(private employeeService: EmployeeService) {}

  ngOnInit(): void {
    this.employeeService.getEmployees()
      .subscribe(employees => this.employees = employees);
  }
}

// In TESTS — inject mock instead of real service
TestBed.configureTestingModule({
  providers: [
    {
      provide: EmployeeService,
      useValue: { getEmployees: () => of([{ id: 1, name: 'Test' }]) }
    }
  ]
});
// Component code unchanged — just gets mock service
```

**One-liner:**
> *"DI lets Angular manage service creation and sharing. Components declare what they need, Angular provides it. Enables singleton services, loose coupling, and easy testing with mocks."*

---

### Q12: What is providedIn: 'root' vs providing in a module vs component?

**✅ Complete Answer:**

```typescript
// 1. providedIn: 'root' — SINGLETON for entire app (most common)
@Injectable({ providedIn: 'root' })
export class GlobalAuthService {
  // ONE instance shared across entire application
  // Tree-shakeable — only included if actually used
}

// 2. Provided in a specific MODULE — singleton within that module
@NgModule({
  providers: [ReportService]  // available only in this module
})
export class ReportsModule {}

// 3. Provided in a COMPONENT — new instance per component
@Component({
  selector: 'app-form',
  providers: [FormStateService]  // NEW instance for each form component
})
export class FormComponent {
  // Each form gets its own FormStateService
  // Children of this component share this same instance
  constructor(private formState: FormStateService) {}
}
// Use case: wizard/multi-step form — each step needs isolated state
```

**One-liner:**
> *"providedIn root = one instance for whole app. Module providers = one instance per module. Component providers = new instance per component. Use component providers when you need isolated state per component instance."*

---

## 🔷 SECTION 4 — HTTP INTERCEPTORS

### Q13: What is an HTTP Interceptor and what can you use it for?

**✅ Complete Answer:**

> *"HTTP Interceptors are middleware for Angular's HttpClient — they intercept every outgoing request and every incoming response globally. You register them once and they run automatically for every HTTP call in your application. Most common uses: adding authorization headers so no component has to manually set them, centralised error handling for 401/403/500 responses, loading spinners that show on any request, and retry logic for failed requests.*
>
> *The key thing to remember: HttpRequest is IMMUTABLE. You cannot modify a request directly — you must CLONE it with the changes you want."*

```typescript
// 1. AUTH INTERCEPTOR — adds token to every request
@Injectable()
export class AuthInterceptor implements HttpInterceptor {
  constructor(private authService: AuthService) {}

  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    const token = this.authService.getToken();

    // Clone request with Authorization header (immutable — must clone)
    const authReq = token
      ? req.clone({
          headers: req.headers.set('Authorization', `Bearer ${token}`)
        })
      : req;

    return next.handle(authReq);
  }
}

// 2. ERROR INTERCEPTOR — handle all HTTP errors centrally
@Injectable()
export class ErrorInterceptor implements HttpInterceptor {
  constructor(private router: Router, private notify: NotificationService) {}

  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    return next.handle(req).pipe(
      catchError((error: HttpErrorResponse) => {
        switch (error.status) {
          case 401:
            this.router.navigate(['/login']); break;
          case 403:
            this.router.navigate(['/unauthorized']); break;
          case 500:
            this.notify.error('Server error — please try again'); break;
        }
        return throwError(() => error);
      })
    );
  }
}

// 3. LOADING INTERCEPTOR — show/hide spinner automatically
@Injectable()
export class LoadingInterceptor implements HttpInterceptor {
  private activeRequests = 0;
  constructor(private loadingService: LoadingService) {}

  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    if (this.activeRequests === 0) this.loadingService.show();
    this.activeRequests++;

    return next.handle(req).pipe(
      finalize(() => {
        this.activeRequests--;
        if (this.activeRequests === 0) this.loadingService.hide();
      })
    );
  }
}

// REGISTER ALL — order matters!
// app.module.ts
providers: [
  { provide: HTTP_INTERCEPTORS, useClass: AuthInterceptor,    multi: true },
  { provide: HTTP_INTERCEPTORS, useClass: ErrorInterceptor,   multi: true },
  { provide: HTTP_INTERCEPTORS, useClass: LoadingInterceptor, multi: true },
]
// Request order:  Auth → Error → Loading → Server
// Response order: Loading → Error → Auth → Component (REVERSE)
```

**One-liner:**
> *"Interceptors are HTTP middleware — run for every request/response globally. Clone the request before modifying (immutable). Register with multi:true. Run in registration order on request, reverse on response."*

---

## 🔷 SECTION 5 — ROUTE GUARDS

### Q14: What are route guards and what types exist?

**✅ Complete Answer:**

```typescript
// CANGUARDS SUMMARY:
// CanActivate      → Can user ENTER this route?
// CanDeactivate    → Can user LEAVE this route? (unsaved changes)
// CanLoad          → Can lazy module be DOWNLOADED?
// CanActivateChild → Can user enter CHILD routes?
// Resolve          → PRE-FETCH data before route activates

// 1. CANACTIVATE — most common
@Injectable({ providedIn: 'root' })
export class AuthGuard implements CanActivate {
  constructor(private auth: AuthService, private router: Router) {}

  canActivate(route: ActivatedRouteSnapshot): boolean | UrlTree {
    if (this.auth.isLoggedIn()) return true;
    return this.router.createUrlTree(['/login'], {
      queryParams: { returnUrl: route.url.join('/') }
    });
  }
}

// Role-based guard
@Injectable({ providedIn: 'root' })
export class RoleGuard implements CanActivate {
  canActivate(route: ActivatedRouteSnapshot): boolean {
    const requiredRole = route.data['role'];
    return this.auth.hasRole(requiredRole);
  }
}

// Route with multiple guards
{
  path: 'admin',
  component: AdminComponent,
  canActivate: [AuthGuard, RoleGuard],
  data: { role: 'ADMIN' }
}
```

---

### Q15: What is CanDeactivate and when do you use it?

**✅ Complete Answer:**

> *"CanDeactivate runs when a user tries to LEAVE a route — pressing browser back, clicking a link, or navigating away. The most common use case is protecting unsaved form changes: 'You have unsaved changes. Are you sure you want to leave?' Without CanDeactivate, users accidentally lose their work by clicking the wrong button. The guard calls a method on the component itself — the component decides whether navigation should be allowed."*

```typescript
// Interface for components that need this guard
export interface CanComponentDeactivate {
  canDeactivate(): boolean | Observable<boolean>;
}

// Guard
@Injectable({ providedIn: 'root' })
export class UnsavedChangesGuard implements CanDeactivate<CanComponentDeactivate> {
  canDeactivate(component: CanComponentDeactivate): boolean | Observable<boolean> {
    return component.canDeactivate();
  }
}

// Component implements the check
@Component({...})
export class EmployeeFormComponent implements CanComponentDeactivate {
  employeeForm: FormGroup;
  isSaved = false;

  canDeactivate(): boolean | Observable<boolean> {
    if (this.employeeForm.pristine || this.isSaved) {
      return true; // form not touched or already saved — allow navigation
    }
    // Ask user for confirmation
    return confirm('You have unsaved changes. Leave anyway?');
    // OR use a modal dialog returning Observable<boolean>
  }
}

// Route config
{
  path: 'employees/:id/edit',
  component: EmployeeFormComponent,
  canDeactivate: [UnsavedChangesGuard]
}
```

**One-liner:**
> *"CanDeactivate guards against accidentally leaving a route. Component implements canDeactivate() method returning boolean. Most common use: unsaved form changes warning."*

---

### Q16: What is the difference between CanActivate and CanLoad?

**✅ Complete Answer:**

> *"CanActivate prevents navigation to a route but the module's JavaScript bundle may already be downloaded to the browser. CanLoad goes one step further — it prevents the lazy-loaded module's JavaScript from being downloaded at all. For a banking application with an admin module, you don't want unauthorized users to even download the admin code — they might find security vulnerabilities in it. So I'd use CanLoad for any module behind authentication, and CanActivate for route-level checks within already-loaded modules."*

```typescript
// CanLoad — prevents JS download
@Injectable({ providedIn: 'root' })
export class CanLoadGuard implements CanLoad {
  constructor(private auth: AuthService) {}

  canLoad(): boolean {
    return this.auth.isLoggedIn();
    // false = admin.module.js NEVER downloaded
  }
}

// Route
{
  path: 'admin',
  loadChildren: () => import('./admin/admin.module').then(m => m.AdminModule),
  canLoad: [CanLoadGuard]     // prevents download
  canActivate: [AuthGuard]    // prevents navigation (additional check)
}
```

**One-liner:**
> *"CanActivate prevents navigation but module may already be downloaded. CanLoad prevents the module JavaScript from downloading at all — better security for protected features."*

---

### Q17: What is a Resolve guard and why use it?

**✅ Complete Answer:**

```typescript
// Resolve — pre-fetches data before route activates
// Without Resolve: component shows loading spinner while fetching
// With Resolve: data ready when component initialises — no loading state needed

@Injectable({ providedIn: 'root' })
export class EmployeeResolver implements Resolve<Employee> {
  constructor(
    private employeeService: EmployeeService,
    private router: Router
  ) {}

  resolve(route: ActivatedRouteSnapshot): Observable<Employee> {
    const id = +route.paramMap.get('id');

    return this.employeeService.getEmployee(id).pipe(
      catchError(() => {
        // Employee not found — redirect to 404
        this.router.navigate(['/not-found']);
        return EMPTY; // complete without emitting — route never activates
      })
    );
  }
}

// Route config
{
  path: 'employees/:id',
  component: EmployeeDetailComponent,
  resolve: { employee: EmployeeResolver }
}

// Component — data pre-loaded, no HTTP call needed
export class EmployeeDetailComponent implements OnInit {
  employee: Employee;

  constructor(private route: ActivatedRoute) {}

  ngOnInit(): void {
    // Data already fetched — available immediately
    this.employee = this.route.snapshot.data['employee'];
    // No loading spinner, no async, no subscribe needed
  }
}
```

**One-liner:**
> *"Resolve pre-fetches data before the route activates — component gets data immediately with no loading state. If fetch fails, redirect to error page and route never activates."*

---

## 🔷 SECTION 6 — RXJS

### Q18: What is an Observable and how is it different from a Promise?

**✅ Complete Answer:**

```
Promise vs Observable:
┌────────────────┬────────────────────┬────────────────────────┐
│ Feature        │ Promise            │ Observable             │
├────────────────┼────────────────────┼────────────────────────┤
│ Values         │ Single value       │ Stream of values       │
│ Execution      │ Eager (starts now) │ Lazy (starts on sub)   │
│ Cancellable    │ No                 │ Yes (unsubscribe)      │
│ Operators      │ .then, .catch      │ map, filter, switchMap │
│ HTTP retry     │ Manual             │ retry() operator       │
│ Cancel HTTP    │ No                 │ Yes (unsubscribe)      │
└────────────────┴────────────────────┴────────────────────────┘

Analogy:
Promise = ordering ONE pizza — get it once, done
Observable = Netflix subscription — keeps streaming new shows
```

```typescript
// Observable types
import { Observable, of, from, interval, Subject, BehaviorSubject } from 'rxjs';

// of — emit fixed values then complete
of(1, 2, 3).subscribe(val => console.log(val)); // 1, 2, 3

// from — convert array/promise to observable
from([10, 20, 30]).subscribe(val => console.log(val));
from(fetch('/api/data')).subscribe(res => console.log(res));

// interval — emit every N milliseconds
interval(1000).subscribe(n => console.log(n)); // 0, 1, 2, 3...

// HTTP — most common in Angular
this.http.get<Employee[]>('/api/employees')
  .subscribe(employees => this.employees = employees);
```

**One-liner:**
> *"Observable is a lazy stream of multiple values — nothing happens until you subscribe. Promise is eager and single-value. Observable supports cancellation, multiple operators, and retry — essential for Angular HTTP."*

---

### Q19: Explain the key RxJS operators — switchMap, mergeMap, concatMap

**✅ Complete Answer:**

```typescript
// switchMap — CANCEL previous, use only LATEST
// Use case: search box — cancel old request when user types again
this.searchInput.valueChanges.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  switchMap(query => this.searchService.search(query))
  // If user types "a", "ab", "abc" quickly:
  // Cancels search for "a", cancels search for "ab"
  // Only returns results for "abc"
).subscribe(results => this.results = results);

// mergeMap — run ALL in PARALLEL, don't cancel
// Use case: delete multiple items — don't cancel previous deletes
ids$.pipe(
  mergeMap(id => this.employeeService.delete(id))
  // All delete requests run simultaneously
  // Results come back in any order
).subscribe(result => console.log('Deleted:', result));

// concatMap — run in SEQUENCE, wait for each to complete
// Use case: sequential operations where order matters
actions$.pipe(
  concatMap(action => this.processAction(action))
  // Action 1 completes, THEN action 2 starts, THEN action 3
  // Order guaranteed
).subscribe();

// exhaustMap — IGNORE new while current is in progress
// Use case: form submit — ignore button clicks during submission
this.submitButton.clicks$.pipe(
  exhaustMap(() => this.formService.submit(this.form.value))
  // If user clicks submit 5 times rapidly
  // Only first click processed
  // Others IGNORED until first completes
).subscribe();
```

**One-liner:**
> *"switchMap: cancel previous, use latest (search). mergeMap: run all parallel (batch operations). concatMap: sequential order guaranteed. exhaustMap: ignore new while current runs (form submit)."*

---

### Q20: What is the difference between Subject, BehaviorSubject, ReplaySubject?

**✅ Complete Answer:**

```typescript
import { Subject, BehaviorSubject, ReplaySubject } from 'rxjs';

// SUBJECT — no initial value, no replay
const subject = new Subject<string>();
subject.subscribe(val => console.log('Sub 1:', val));
subject.next('Hello');       // Sub 1: Hello
// New subscriber AFTER emission:
subject.subscribe(val => console.log('Sub 2:', val));
subject.next('World');       // Sub 1: World, Sub 2: World
// Sub 2 MISSED 'Hello' — joined too late

// BEHAVIORSUBJECT — has initial value, new subscribers get CURRENT value
const behaviorSub = new BehaviorSubject<string>('initial');
behaviorSub.subscribe(val => console.log('Sub 1:', val)); // 'initial'
behaviorSub.next('updated');
// New subscriber gets CURRENT value immediately
behaviorSub.subscribe(val => console.log('Sub 2:', val)); // 'updated'
// Most used for: current user, cart, theme, app state

// REPLAYSUBJECT — replays last N values to new subscribers
const replaySub = new ReplaySubject<string>(3); // replay last 3
replaySub.next('msg1');
replaySub.next('msg2');
replaySub.next('msg3');
replaySub.next('msg4');
// New subscriber gets: msg2, msg3, msg4 (last 3)
replaySub.subscribe(val => console.log(val));
// Use for: event logs, notification history, late joiners

// In real Angular service (most common pattern):
@Injectable({ providedIn: 'root' })
export class AuthService {
  private currentUser = new BehaviorSubject<User>(null);
  currentUser$ = this.currentUser.asObservable(); // expose as Observable only

  login(user: User): void {
    this.currentUser.next(user); // update
  }

  logout(): void {
    this.currentUser.next(null);
  }
}
```

**One-liner:**
> *"Subject: no history, no initial value. BehaviorSubject: requires initial value, new subscribers get current value immediately — use for shared state. ReplaySubject: replays last N values — use for event history."*

---

### Q21: How do you prevent memory leaks in Angular?

**✅ Complete Answer:**

```typescript
// PROBLEM: Subscriptions don't auto-clean up
// If component is destroyed but subscription lives — memory leak!

// METHOD 1: Manual unsubscribe in ngOnDestroy
export class EmployeeComponent implements OnInit, OnDestroy {
  private subscription: Subscription;

  ngOnInit(): void {
    this.subscription = this.employeeService.getEmployees()
      .subscribe(employees => this.employees = employees);
  }

  ngOnDestroy(): void {
    this.subscription.unsubscribe(); // clean up
  }
}

// METHOD 2: takeUntil pattern (best for multiple subscriptions)
export class EmployeeComponent implements OnInit, OnDestroy {
  private destroy$ = new Subject<void>();

  ngOnInit(): void {
    this.employeeService.getEmployees()
      .pipe(takeUntil(this.destroy$))  // auto-complete when destroy$ emits
      .subscribe(employees => this.employees = employees);

    this.searchService.results$
      .pipe(takeUntil(this.destroy$))  // same destroy$ handles multiple
      .subscribe(results => this.results = results);
  }

  ngOnDestroy(): void {
    this.destroy$.next();     // trigger completion for all takeUntil subscribers
    this.destroy$.complete(); // complete the subject itself
  }
}

// METHOD 3: Async Pipe — BEST (auto-subscribe AND auto-unsubscribe)
@Component({
  template: `
    <div *ngFor="let emp of employees$ | async">
      {{ emp.name }}
    </div>
  `
})
export class EmployeeComponent {
  employees$ = this.employeeService.getEmployees(); // Observable, not subscribed yet
  // async pipe subscribes on init, unsubscribes on destroy automatically
  // Also works with OnPush change detection!
}
```

**One-liner:**
> *"Three ways: unsubscribe manually in ngOnDestroy, use takeUntil with a destroy$ Subject for multiple subscriptions, or use async pipe in template which auto-subscribes and auto-unsubscribes."*

---

## 🔷 SECTION 7 — FORMS

### Q22: Template-driven vs Reactive Forms — when to use each?

**✅ Complete Answer:**

```typescript
// TEMPLATE-DRIVEN — simple forms, logic in HTML
// Requires: FormsModule

<form #empForm="ngForm" (ngSubmit)="onSubmit(empForm)">
  <input
    name="name"
    [(ngModel)]="employee.name"
    required
    minlength="3"
    #nameField="ngModel">

  <div *ngIf="nameField.invalid && nameField.touched">
    <span *ngIf="nameField.errors?.['required']">Required</span>
    <span *ngIf="nameField.errors?.['minlength']">Min 3 chars</span>
  </div>

  <button [disabled]="empForm.invalid">Save</button>
</form>

// REACTIVE FORMS — complex forms, logic in TypeScript
// Requires: ReactiveFormsModule
export class EmployeeFormComponent implements OnInit {
  form: FormGroup;

  constructor(private fb: FormBuilder) {}

  ngOnInit(): void {
    this.form = this.fb.group({
      name:  ['', [Validators.required, Validators.minLength(3)]],
      email: ['', [Validators.required, Validators.email]],
      address: this.fb.group({
        city:   ['', Validators.required],
        street: ['']
      }),
      skills: this.fb.array([])  // dynamic list
    });

    // React to value changes as Observable
    this.form.get('name').valueChanges
      .pipe(debounceTime(300))
      .subscribe(value => console.log('Name changed:', value));
  }

  get nameControl() { return this.form.get('name'); }
  get skills() { return this.form.get('skills') as FormArray; }

  addSkill(): void {
    this.skills.push(this.fb.control('', Validators.required));
  }

  onSubmit(): void {
    if (this.form.valid) {
      console.log(this.form.value); // typed form values
    }
  }
}
```

**When to use which:**
```
Template-driven:  Simple contact forms, login forms, quick prototypes
Reactive:         Complex validation, dynamic fields (FormArray),
                  unit testable forms, subscribe to value changes,
                  programmatic form control — ALWAYS for enterprise
```

**One-liner:**
> *"Template-driven: simple, logic in HTML, less code. Reactive: complex validation, dynamic fields, fully testable, programmatic control. For enterprise apps always use Reactive."*

---

## 🔷 SECTION 8 — PERFORMANCE

### Q23: What is OnPush change detection and when should you use it?

**✅ Complete Answer:**

> *"By default, Angular checks every component in the tree after ANY browser event — a click, a keypress, an HTTP response. For a large application with hundreds of components, this is expensive. OnPush strategy tells Angular: only check this component when its @Input() REFERENCE changes, when an event is emitted FROM this component, or when an async pipe resolves. This dramatically reduces unnecessary checking.*
>
> *The critical thing with OnPush: you must use immutable data patterns. You cannot mutate an existing object and expect Angular to detect it — you must create a new object reference."*

```typescript
@Component({
  selector: 'app-employee-list',
  template: `<div *ngFor="let emp of employees">{{ emp.name }}</div>`,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class EmployeeListComponent {
  @Input() employees: Employee[];
}

// PARENT — must pass NEW reference to trigger OnPush detection
// BAD — mutation, OnPush won't detect:
this.employees.push(newEmployee);        // modifying existing array
this.employees[0].name = 'New Name';     // modifying existing object

// GOOD — new reference triggers OnPush:
this.employees = [...this.employees, newEmployee];  // new array
this.employees = this.employees.map(e =>            // new array
  e.id === id ? { ...e, name: 'New Name' } : e
);

// Manual trigger if needed
export class MyComponent {
  constructor(private cdr: ChangeDetectorRef) {}

  updateFromExternalSource(): void {
    // Data came from outside Angular (WebSocket, etc.)
    this.cdr.markForCheck(); // schedule check for next cycle
  }
}
```

**One-liner:**
> *"OnPush only checks component when Input reference changes, component emits event, or async pipe resolves. Must use immutable data — spread to create new references. Dramatic performance improvement for large lists."*

---

### Q24: What is lazy loading and how does it improve performance?

**✅ Complete Answer:**

```typescript
// Without lazy loading: entire app downloaded upfront
// PROBLEM: User visits home page but downloads admin, reports, settings code
// All 5MB of JavaScript loaded even if user never visits those routes

// With lazy loading: feature module downloaded only when route accessed
// app-routing.module.ts
const routes: Routes = [
  { path: '', component: HomeComponent },  // eagerly loaded
  {
    path: 'reports',
    loadChildren: () =>                    // lazily loaded
      import('./reports/reports.module').then(m => m.ReportsModule)
    // reports.module.js only downloads when user navigates to /reports
  },
  {
    path: 'admin',
    loadChildren: () =>
      import('./admin/admin.module').then(m => m.AdminModule),
    canLoad: [AuthGuard]   // don't even download if not authorized
  }
];

// Preloading strategy — best of both worlds
// Download lazy modules in BACKGROUND after initial load
@NgModule({
  imports: [RouterModule.forRoot(routes, {
    preloadingStrategy: PreloadAllModules
    // Initial load fast (small bundle)
    // Background download of lazy modules
    // Navigation feels instant after initial load
  })]
})
export class AppRoutingModule {}
```

**One-liner:**
> *"Lazy loading splits the app into chunks downloaded only when needed. Reduces initial bundle size — faster first load. PreloadAllModules downloads lazy chunks in background after initial load — best UX."*

---

## 🔷 SECTION 9 — MODERN ANGULAR

### Q25: What are Angular Signals and how do they differ from RxJS?

**✅ Complete Answer:**

```typescript
import { signal, computed, effect } from '@angular/core';

@Component({
  selector: 'app-employee',
  template: `
    <h2>{{ fullName() }}</h2>
    <p>Salary: {{ formattedSalary() }}</p>
    <button (click)="giveRaise()">Give 10% Raise</button>
  `
})
export class EmployeeComponent {
  // Writable signal — like reactive useState
  firstName = signal('Abhishek');
  salary = signal(50000);

  // Computed — auto-updates when dependencies change
  fullName = computed(() => `${this.firstName()} Kumar`);
  formattedSalary = computed(() => `₹${this.salary().toLocaleString()}`);

  // Effect — side effects when signals change
  constructor() {
    effect(() => {
      console.log('Salary updated to:', this.salary()); // auto-tracks
      localStorage.setItem('salary', String(this.salary()));
    });
  }

  giveRaise(): void {
    this.salary.update(current => current * 1.1); // update based on current
    // OR: this.salary.set(55000); // set absolute value
  }
}
```

**Signals vs RxJS:**
```
Signals:
✅ Synchronous — read value instantly by calling it
✅ Simple — no subscribe, no unsubscribe, no memory leaks
✅ Automatic dependency tracking in computed/effect
✅ Fine-grained reactivity — only affected components update
✅ Great for: local UI state, derived values, template bindings

RxJS:
✅ Asynchronous streams — time-based, event-based
✅ Powerful operators (switchMap, debounceTime, etc.)
✅ Great for: HTTP calls, WebSockets, complex async flows
✅ Cancellable HTTP requests

Use BOTH: Signals for local state, RxJS for async operations
```

**One-liner:**
> *"Signals are Angular's synchronous reactive primitive — read by calling them like a function, computed auto-tracks dependencies, effects run on change. RxJS is still better for async operations. In modern Angular you use both."*

---

### Q26: What are Standalone Components in Angular 14+?

**✅ Complete Answer:**

```typescript
// OLD WAY — must declare in NgModule
@NgModule({
  declarations: [EmployeeCardComponent],
  imports: [CommonModule],
  exports: [EmployeeCardComponent]
})
export class EmployeeModule {}

// NEW WAY — standalone, no module needed
@Component({
  selector: 'app-employee-card',
  standalone: true,                           // self-contained
  imports: [CommonModule, RouterModule, AsyncPipe],  // import directly
  template: `
    <div>{{ employee.name }}</div>
    <div *ngFor="let skill of employee.skills">{{ skill }}</div>
  `
})
export class EmployeeCardComponent {
  @Input() employee: Employee;
}

// Bootstrap standalone app (no AppModule needed)
// main.ts
bootstrapApplication(AppComponent, {
  providers: [
    provideRouter(routes),
    provideHttpClient(),
    importProvidersFrom(FormsModule)
  ]
});
```

**One-liner:**
> *"Standalone components are self-contained — they import their own dependencies directly without NgModule. Angular 14+ feature that simplifies architecture. New apps should use standalone by default."*

---

## 🔷 SECTION 10 — TESTING

### Q27: How do you unit test an Angular component?

**✅ Complete Answer:**

```typescript
// employee-card.component.spec.ts
describe('EmployeeCardComponent', () => {
  let component: EmployeeCardComponent;
  let fixture: ComponentFixture<EmployeeCardComponent>;
  let mockEmployeeService: jasmine.SpyObj<EmployeeService>;

  beforeEach(async () => {
    // Create mock service
    mockEmployeeService = jasmine.createSpyObj('EmployeeService', ['getEmployees']);
    mockEmployeeService.getEmployees.and.returnValue(of([
      { id: 1, name: 'Abhishek', salary: 50000 }
    ]));

    await TestBed.configureTestingModule({
      declarations: [EmployeeCardComponent],
      providers: [
        { provide: EmployeeService, useValue: mockEmployeeService }
      ]
    }).compileComponents();

    fixture = TestBed.createComponent(EmployeeCardComponent);
    component = fixture.componentInstance;
  });

  it('should display employee name', () => {
    component.employee = { id: 1, name: 'Abhishek', salary: 50000 };
    fixture.detectChanges(); // trigger change detection

    const nameEl: HTMLElement = fixture.nativeElement.querySelector('h3');
    expect(nameEl.textContent).toContain('Abhishek');
  });

  it('should emit employeeSelected when button clicked', () => {
    component.employee = { id: 1, name: 'Abhishek', salary: 50000 };
    fixture.detectChanges();

    spyOn(component.employeeSelected, 'emit');

    const button: HTMLButtonElement = fixture.nativeElement.querySelector('button');
    button.click();

    expect(component.employeeSelected.emit)
      .toHaveBeenCalledWith(component.employee);
  });

  it('should hide salary when showSalary is false', () => {
    component.employee = { id: 1, name: 'Abhishek', salary: 50000 };
    component.showSalary = false;
    fixture.detectChanges();

    const salaryEl = fixture.nativeElement.querySelector('.salary');
    expect(salaryEl).toBeNull(); // element should not exist
  });
});
```

**One-liner:**
> *"TestBed creates a testing module. Mock services with jasmine.createSpyObj. fixture.detectChanges triggers change detection. Access DOM via fixture.nativeElement. Spy on EventEmitters to test @Output emissions."*

---

## ⚡ QUICK REFERENCE — ALL ONE-LINERS

```
DATA BINDING:
  {{ }}          Interpolation — display values (component → template)
  [ ]            Property binding — set DOM property (component → template)
  ( )            Event binding — listen to events (template → component)
  [( )]          Two-way binding — both directions (requires FormsModule)

COMPONENT COMMUNICATION:
  @Input()              Parent → Child (property binding)
  @Output() + EventEmitter   Child → Parent (event binding)
  @ViewChild            Parent accesses child directly (AfterViewInit only)
  Shared Service + BehaviorSubject   Sibling ↔ Sibling

LIFECYCLE ORDER (Key 4):
  OnChanges → OnInit → AfterViewInit → OnDestroy
  Constructor = DI only. OnInit = everything else.

ROUTE GUARDS:
  CanActivate     = can user enter?
  CanDeactivate   = can user leave? (unsaved changes)
  CanLoad         = can JS download? (lazy + security)
  Resolve         = pre-fetch data before route

INTERCEPTORS:
  Order: Auth → Error → Loading → Server (request)
  Reverse on response
  Always clone request — HttpRequest is immutable

RXJS OPERATORS:
  switchMap    = cancel previous, latest wins (search)
  mergeMap     = all parallel, don't cancel (batch)
  concatMap    = sequential, ordered (step by step)
  exhaustMap   = ignore new while running (form submit)
  takeUntil    = auto-unsubscribe on destroy
  debounceTime = wait after last emission
  distinctUntilChanged = only if value changed

MEMORY LEAKS:
  async pipe     = auto-subscribe + auto-unsubscribe (best)
  takeUntil      = multiple subscriptions, one destroy$
  unsubscribe()  = manual, in ngOnDestroy

FORMS:
  Template-driven = simple, logic in HTML, ngModel
  Reactive        = complex, logic in TypeScript, testable, dynamic

PERFORMANCE:
  OnPush           = only check on Input reference change
  trackBy          = prevent full DOM re-render in ngFor
  lazy loading     = download module only when needed
  async pipe       = compatible with OnPush
  Signals          = fine-grained, synchronous reactivity (Angular 16+)

DIRECTIVES:
  Structural (*ngIf, *ngFor) = add/remove DOM elements
  Attribute ([ngClass])      = change appearance
  Custom                     = your own behaviour

CHANGE DETECTION:
  Default = check entire tree on every async event
  OnPush  = check ONLY when Input ref changes / event / async pipe
  Rule    = always pass new object/array reference with OnPush

MODERN ANGULAR:
  Standalone = no NgModule needed (Angular 14+)
  Signals    = synchronous reactive state (Angular 16+)
  @if/@for   = new control flow syntax (Angular 17+)
```

---

## 📅 SELF-ASSESSMENT TRACKER

| Question | Topic | Session 1 | Session 2 | Session 3 | Ready? |
|---|---|---|---|---|---|
| Q1  | Angular vs React | ⬜ | ⬜ | ⬜ | ⬜ |
| Q2  | 4 data binding types | ⬜ | ⬜ | ⬜ | ⬜ |
| Q3  | Lifecycle hooks order | ⬜ | ⬜ | ⬜ | ⬜ |
| Q4  | Constructor vs ngOnInit | ⬜ | ⬜ | ⬜ | ⬜ |
| Q5  | 3 directive types | ⬜ | ⬜ | ⬜ | ⬜ |
| Q6  | Parent → Child @Input | ⬜ | ⬜ | ⬜ | ⬜ |
| Q7  | Child → Parent @Output | ⬜ | ⬜ | ⬜ | ⬜ |
| Q8  | @ViewChild | ⬜ | ⬜ | ⬜ | ⬜ |
| Q9  | Sibling via service | ⬜ | ⬜ | ⬜ | ⬜ |
| Q10 | ng-content projection | ⬜ | ⬜ | ⬜ | ⬜ |
| Q11 | Dependency Injection | ⬜ | ⬜ | ⬜ | ⬜ |
| Q12 | providedIn scope | ⬜ | ⬜ | ⬜ | ⬜ |
| Q13 | HTTP Interceptors | ⬜ | ⬜ | ⬜ | ⬜ |
| Q14 | Route guards overview | ⬜ | ⬜ | ⬜ | ⬜ |
| Q15 | CanDeactivate | ⬜ | ⬜ | ⬜ | ⬜ |
| Q16 | CanActivate vs CanLoad | ⬜ | ⬜ | ⬜ | ⬜ |
| Q17 | Resolve guard | ⬜ | ⬜ | ⬜ | ⬜ |
| Q18 | Observable vs Promise | ⬜ | ⬜ | ⬜ | ⬜ |
| Q19 | switchMap vs mergeMap | ⬜ | ⬜ | ⬜ | ⬜ |
| Q20 | Subject vs BehaviorSubject | ⬜ | ⬜ | ⬜ | ⬜ |
| Q21 | Prevent memory leaks | ⬜ | ⬜ | ⬜ | ⬜ |
| Q22 | Template vs Reactive forms | ⬜ | ⬜ | ⬜ | ⬜ |
| Q23 | OnPush change detection | ⬜ | ⬜ | ⬜ | ⬜ |
| Q24 | Lazy loading | ⬜ | ⬜ | ⬜ | ⬜ |
| Q25 | Signals vs RxJS | ⬜ | ⬜ | ⬜ | ⬜ |
| Q26 | Standalone components | ⬜ | ⬜ | ⬜ | ⬜ |
| Q27 | Unit testing components | ⬜ | ⬜ | ⬜ | ⬜ |

**Rating: ⬜ = not tried | ❌ = wrong | 🟡 = partial | ✅ = nailed it**

---

*Angular mastery = knowing WHY, not just HOW.*
*In interviews: always explain trade-offs — that separates senior engineers.*