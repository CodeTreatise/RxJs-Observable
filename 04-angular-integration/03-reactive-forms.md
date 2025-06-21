# Reactive Forms with RxJS

## Overview

Angular Reactive Forms and RxJS create a powerful combination for building dynamic, responsive forms with complex validation, real-time updates, and sophisticated user interactions. This lesson covers comprehensive patterns for integrating RxJS with Angular Reactive Forms, including custom validators, dynamic forms, cross-field validation, and advanced form state management.

## Reactive Forms Fundamentals with RxJS

### Basic Form Setup with Observables

**Simple Reactive Form:**
```typescript
import { Component, OnInit, OnDestroy } from '@angular/core';
import { FormBuilder, FormGroup, Validators, AbstractControl } from '@angular/forms';
import { Subject, Observable, combineLatest } from 'rxjs';
import { map, startWith, distinctUntilChanged, takeUntil, debounceTime } from 'rxjs/operators';

@Component({
  selector: 'app-user-form',
  template: `
    <form [formGroup]="userForm" (ngSubmit)="onSubmit()">
      <div class="form-field">
        <label>Email</label>
        <input 
          formControlName="email" 
          type="email"
          [class.error]="emailInvalid$ | async">
        <div *ngIf="emailError$ | async as error" class="error-message">
          {{ error }}
        </div>
      </div>

      <div class="form-field">
        <label>Password</label>
        <input 
          formControlName="password" 
          type="password"
          [class.error]="passwordInvalid$ | async">
        <div *ngIf="passwordError$ | async as error" class="error-message">
          {{ error }}
        </div>
      </div>

      <div class="form-field">
        <label>Confirm Password</label>
        <input 
          formControlName="confirmPassword" 
          type="password"
          [class.error]="confirmPasswordInvalid$ | async">
        <div *ngIf="confirmPasswordError$ | async as error" class="error-message">
          {{ error }}
        </div>
      </div>

      <div class="form-actions">
        <button 
          type="submit" 
          [disabled]="!(formValid$ | async) || (submitting$ | async)">
          {{ (submitting$ | async) ? 'Creating Account...' : 'Create Account' }}
        </button>
      </div>

      <div class="form-debug" *ngIf="showDebug">
        <h4>Form State</h4>
        <pre>{{ formState$ | async | json }}</pre>
      </div>
    </form>
  `,
  styleUrls: ['./user-form.component.scss']
})
export class UserFormComponent implements OnInit, OnDestroy {
  private destroy$ = new Subject<void>();
  
  userForm: FormGroup;
  submitting$ = new Subject<boolean>();
  showDebug = false;

  // Form state observables
  formValid$: Observable<boolean>;
  formState$: Observable<any>;
  
  // Field-specific observables
  emailInvalid$: Observable<boolean>;
  emailError$: Observable<string>;
  passwordInvalid$: Observable<boolean>;
  passwordError$: Observable<string>;
  confirmPasswordInvalid$: Observable<boolean>;
  confirmPasswordError$: Observable<string>;

  constructor(
    private fb: FormBuilder,
    private userService: UserService
  ) {
    this.buildForm();
    this.setupFormObservables();
  }

  ngOnInit() {
    // Auto-save draft
    this.userForm.valueChanges.pipe(
      debounceTime(1000),
      distinctUntilChanged(),
      takeUntil(this.destroy$)
    ).subscribe(formValue => {
      this.saveDraft(formValue);
    });

    // Load saved draft
    this.loadDraft();
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }

  private buildForm(): void {
    this.userForm = this.fb.group({
      email: ['', [Validators.required, Validators.email], [this.emailExistsValidator.bind(this)]],
      password: ['', [Validators.required, Validators.minLength(8), this.passwordStrengthValidator]],
      confirmPassword: ['', [Validators.required]]
    }, {
      validators: [this.passwordMatchValidator]
    });
  }

  private setupFormObservables(): void {
    // Overall form state
    this.formValid$ = this.userForm.statusChanges.pipe(
      startWith(this.userForm.status),
      map(status => status === 'VALID')
    );

    this.formState$ = combineLatest([
      this.userForm.valueChanges.pipe(startWith(this.userForm.value)),
      this.userForm.statusChanges.pipe(startWith(this.userForm.status))
    ]).pipe(
      map(([value, status]) => ({
        value,
        status,
        errors: this.userForm.errors,
        touched: this.userForm.touched,
        dirty: this.userForm.dirty
      }))
    );

    // Email field observables
    const emailControl = this.userForm.get('email')!;
    this.emailInvalid$ = combineLatest([
      emailControl.statusChanges.pipe(startWith(emailControl.status)),
      emailControl.valueChanges.pipe(startWith(emailControl.value))
    ]).pipe(
      map(() => emailControl.invalid && emailControl.touched)
    );

    this.emailError$ = this.emailInvalid$.pipe(
      map(isInvalid => {
        if (!isInvalid) return '';
        const errors = emailControl.errors;
        if (errors?.['required']) return 'Email is required';
        if (errors?.['email']) return 'Please enter a valid email address';
        if (errors?.['emailExists']) return 'This email is already registered';
        return '';
      })
    );

    // Password field observables
    const passwordControl = this.userForm.get('password')!;
    this.passwordInvalid$ = combineLatest([
      passwordControl.statusChanges.pipe(startWith(passwordControl.status)),
      passwordControl.valueChanges.pipe(startWith(passwordControl.value))
    ]).pipe(
      map(() => passwordControl.invalid && passwordControl.touched)
    );

    this.passwordError$ = this.passwordInvalid$.pipe(
      map(isInvalid => {
        if (!isInvalid) return '';
        const errors = passwordControl.errors;
        if (errors?.['required']) return 'Password is required';
        if (errors?.['minlength']) return 'Password must be at least 8 characters';
        if (errors?.['passwordStrength']) return 'Password must contain uppercase, lowercase, number, and special character';
        return '';
      })
    );

    // Confirm password field observables
    const confirmPasswordControl = this.userForm.get('confirmPassword')!;
    this.confirmPasswordInvalid$ = combineLatest([
      confirmPasswordControl.statusChanges.pipe(startWith(confirmPasswordControl.status)),
      this.userForm.statusChanges.pipe(startWith(this.userForm.status))
    ]).pipe(
      map(() => {
        const hasError = this.userForm.errors?.['passwordMismatch'] && confirmPasswordControl.touched;
        return hasError || (confirmPasswordControl.invalid && confirmPasswordControl.touched);
      })
    );

    this.confirmPasswordError$ = this.confirmPasswordInvalid$.pipe(
      map(isInvalid => {
        if (!isInvalid) return '';
        if (confirmPasswordControl.errors?.['required']) return 'Please confirm your password';
        if (this.userForm.errors?.['passwordMismatch']) return 'Passwords do not match';
        return '';
      })
    );
  }

  // Custom validators
  private passwordStrengthValidator(control: AbstractControl): {[key: string]: any} | null {
    const value = control.value;
    if (!value) return null;

    const hasUpperCase = /[A-Z]/.test(value);
    const hasLowerCase = /[a-z]/.test(value);
    const hasNumber = /\d/.test(value);
    const hasSpecialChar = /[!@#$%^&*(),.?":{}|<>]/.test(value);

    const isValid = hasUpperCase && hasLowerCase && hasNumber && hasSpecialChar;
    return isValid ? null : { passwordStrength: true };
  }

  private passwordMatchValidator(group: AbstractControl): {[key: string]: any} | null {
    const password = group.get('password');
    const confirmPassword = group.get('confirmPassword');
    
    if (!password || !confirmPassword) return null;
    
    return password.value === confirmPassword.value ? null : { passwordMismatch: true };
  }

  private emailExistsValidator(control: AbstractControl): Observable<{[key: string]: any} | null> {
    if (!control.value) return of(null);
    
    return this.userService.checkEmailExists(control.value).pipe(
      map(exists => exists ? { emailExists: true } : null),
      catchError(() => of(null))
    );
  }

  onSubmit(): void {
    if (this.userForm.valid) {
      this.submitting$.next(true);
      
      this.userService.createUser(this.userForm.value).pipe(
        takeUntil(this.destroy$),
        finalize(() => this.submitting$.next(false))
      ).subscribe({
        next: (user) => {
          console.log('User created successfully:', user);
          this.userForm.reset();
          this.clearDraft();
        },
        error: (error) => {
          console.error('Failed to create user:', error);
          this.handleSubmissionError(error);
        }
      });
    }
  }

  private saveDraft(formValue: any): void {
    localStorage.setItem('userFormDraft', JSON.stringify(formValue));
  }

  private loadDraft(): void {
    const draft = localStorage.getItem('userFormDraft');
    if (draft) {
      try {
        const draftValue = JSON.parse(draft);
        this.userForm.patchValue(draftValue);
      } catch (error) {
        console.warn('Failed to load form draft:', error);
      }
    }
  }

  private clearDraft(): void {
    localStorage.removeItem('userFormDraft');
  }

  private handleSubmissionError(error: any): void {
    if (error.status === 422 && error.error.validation_errors) {
      this.applyServerValidationErrors(error.error.validation_errors);
    }
  }

  private applyServerValidationErrors(validationErrors: any): void {
    Object.keys(validationErrors).forEach(field => {
      const control = this.userForm.get(field);
      if (control) {
        control.setErrors({ serverError: validationErrors[field][0] });
      }
    });
  }
}
```

### Dynamic Forms with RxJS

**Dynamic Form Builder:**
```typescript
@Injectable({
  providedIn: 'root'
})
export class DynamicFormService {
  constructor(private fb: FormBuilder) {}

  buildFormFromConfig(config: FormFieldConfig[]): FormGroup {
    const group: any = {};

    config.forEach(field => {
      const validators = this.buildValidators(field.validation || []);
      const asyncValidators = this.buildAsyncValidators(field.asyncValidation || []);
      
      group[field.name] = [field.value || '', validators, asyncValidators];
    });

    return this.fb.group(group);
  }

  private buildValidators(validationConfig: ValidationConfig[]): any[] {
    return validationConfig.map(config => {
      switch (config.type) {
        case 'required':
          return Validators.required;
        case 'email':
          return Validators.email;
        case 'minLength':
          return Validators.minLength(config.value);
        case 'maxLength':
          return Validators.maxLength(config.value);
        case 'pattern':
          return Validators.pattern(config.value);
        case 'min':
          return Validators.min(config.value);
        case 'max':
          return Validators.max(config.value);
        default:
          return null;
      }
    }).filter(validator => validator !== null);
  }

  private buildAsyncValidators(asyncValidationConfig: AsyncValidationConfig[]): any[] {
    return asyncValidationConfig.map(config => {
      switch (config.type) {
        case 'uniqueEmail':
          return this.uniqueEmailValidator.bind(this);
        case 'usernameAvailable':
          return this.usernameAvailableValidator.bind(this);
        default:
          return null;
      }
    }).filter(validator => validator !== null);
  }

  private uniqueEmailValidator(control: AbstractControl): Observable<{[key: string]: any} | null> {
    if (!control.value) return of(null);
    
    return timer(300).pipe(
      switchMap(() => 
        // Simulate API call
        of(control.value === 'taken@example.com').pipe(
          map(exists => exists ? { emailTaken: true } : null)
        )
      )
    );
  }

  private usernameAvailableValidator(control: AbstractControl): Observable<{[key: string]: any} | null> {
    if (!control.value) return of(null);
    
    return timer(300).pipe(
      switchMap(() => 
        of(control.value === 'admin').pipe(
          map(taken => taken ? { usernameTaken: true } : null)
        )
      )
    );
  }
}

interface FormFieldConfig {
  name: string;
  type: 'text' | 'email' | 'password' | 'number' | 'select' | 'checkbox';
  label: string;
  value?: any;
  options?: { value: any; label: string }[];
  validation?: ValidationConfig[];
  asyncValidation?: AsyncValidationConfig[];
}

interface ValidationConfig {
  type: string;
  value?: any;
  message?: string;
}

interface AsyncValidationConfig {
  type: string;
  message?: string;
}

// Dynamic Form Component
@Component({
  selector: 'app-dynamic-form',
  template: `
    <form [formGroup]="dynamicForm" (ngSubmit)="onSubmit()">
      <div *ngFor="let field of formConfig" class="form-field">
        <label>{{ field.label }}</label>
        
        <!-- Text inputs -->
        <input 
          *ngIf="field.type === 'text' || field.type === 'email' || field.type === 'password'"
          [type]="field.type"
          [formControlName]="field.name"
          [class.error]="isFieldInvalid(field.name) | async">
        
        <!-- Number input -->
        <input 
          *ngIf="field.type === 'number'"
          type="number"
          [formControlName]="field.name"
          [class.error]="isFieldInvalid(field.name) | async">
        
        <!-- Select dropdown -->
        <select 
          *ngIf="field.type === 'select'"
          [formControlName]="field.name"
          [class.error]="isFieldInvalid(field.name) | async">
          <option value="">Select...</option>
          <option *ngFor="let option of field.options" [value]="option.value">
            {{ option.label }}
          </option>
        </select>
        
        <!-- Checkbox -->
        <input 
          *ngIf="field.type === 'checkbox'"
          type="checkbox"
          [formControlName]="field.name">
        
        <!-- Error messages -->
        <div *ngIf="getFieldError(field.name) | async as error" class="error-message">
          {{ error }}
        </div>
      </div>

      <button 
        type="submit" 
        [disabled]="!dynamicForm.valid || (submitting$ | async)">
        {{ (submitting$ | async) ? 'Submitting...' : 'Submit' }}
      </button>
    </form>
  `
})
export class DynamicFormComponent implements OnInit {
  @Input() formConfig: FormFieldConfig[] = [];
  @Output() formSubmit = new EventEmitter<any>();

  dynamicForm: FormGroup;
  submitting$ = new BehaviorSubject(false);

  constructor(private dynamicFormService: DynamicFormService) {}

  ngOnInit() {
    this.dynamicForm = this.dynamicFormService.buildFormFromConfig(this.formConfig);
  }

  isFieldInvalid(fieldName: string): Observable<boolean> {
    const control = this.dynamicForm.get(fieldName);
    if (!control) return of(false);

    return combineLatest([
      control.statusChanges.pipe(startWith(control.status)),
      control.valueChanges.pipe(startWith(control.value))
    ]).pipe(
      map(() => control.invalid && control.touched)
    );
  }

  getFieldError(fieldName: string): Observable<string> {
    const control = this.dynamicForm.get(fieldName);
    if (!control) return of('');

    return this.isFieldInvalid(fieldName).pipe(
      map(isInvalid => {
        if (!isInvalid) return '';
        
        const errors = control.errors;
        if (errors?.['required']) return 'This field is required';
        if (errors?.['email']) return 'Please enter a valid email';
        if (errors?.['minlength']) return `Minimum length is ${errors['minlength'].requiredLength}`;
        if (errors?.['maxlength']) return `Maximum length is ${errors['maxlength'].requiredLength}`;
        if (errors?.['emailTaken']) return 'This email is already registered';
        if (errors?.['usernameTaken']) return 'This username is not available';
        
        return 'Invalid input';
      })
    );
  }

  onSubmit() {
    if (this.dynamicForm.valid) {
      this.submitting$.next(true);
      this.formSubmit.emit(this.dynamicForm.value);
      
      // Reset submitting state after a delay (or when parent component signals completion)
      setTimeout(() => this.submitting$.next(false), 2000);
    }
  }
}
```

### Cross-Field Validation with RxJS

**Advanced Cross-Field Validators:**
```typescript
@Component({
  selector: 'app-booking-form',
  template: `
    <form [formGroup]="bookingForm">
      <div class="form-row">
        <div class="form-field">
          <label>Check-in Date</label>
          <input 
            type="date" 
            formControlName="checkInDate"
            [class.error]="checkInInvalid$ | async">
        </div>
        
        <div class="form-field">
          <label>Check-out Date</label>
          <input 
            type="date" 
            formControlName="checkOutDate"
            [class.error]="checkOutInvalid$ | async">
        </div>
      </div>

      <div class="form-field">
        <label>Number of Guests</label>
        <input 
          type="number" 
          formControlName="guests"
          [class.error]="guestsInvalid$ | async">
      </div>

      <div class="form-field">
        <label>Room Type</label>
        <select formControlName="roomType" [class.error]="roomTypeInvalid$ | async">
          <option value="">Select room type</option>
          <option *ngFor="let room of availableRooms$ | async" [value]="room.id">
            {{ room.name }} - {{ room.price | currency }}
          </option>
        </select>
      </div>

      <!-- Cross-field validation errors -->
      <div *ngIf="dateRangeError$ | async as error" class="error-message">
        {{ error }}
      </div>

      <div *ngIf="capacityError$ | async as error" class="error-message">
        {{ error }}
      </div>

      <div *ngIf="availabilityError$ | async as error" class="error-message">
        {{ error }}
      </div>

      <!-- Total price calculation -->
      <div class="price-summary">
        <h3>Total: {{ totalPrice$ | async | currency }}</h3>
        <p>{{ nightCount$ | async }} nights</p>
      </div>

      <button 
        type="submit" 
        [disabled]="!bookingForm.valid || !(availabilityConfirmed$ | async)"
        (click)="onSubmit()">
        Book Now
      </button>
    </form>
  `
})
export class BookingFormComponent implements OnInit, OnDestroy {
  private destroy$ = new Subject<void>();
  
  bookingForm: FormGroup;

  // Form field observables
  checkInInvalid$: Observable<boolean>;
  checkOutInvalid$: Observable<boolean>;
  guestsInvalid$: Observable<boolean>;
  roomTypeInvalid$: Observable<boolean>;

  // Cross-field validation observables
  dateRangeError$: Observable<string>;
  capacityError$: Observable<string>;
  availabilityError$: Observable<string>;

  // Calculated values
  nightCount$: Observable<number>;
  totalPrice$: Observable<number>;
  availableRooms$: Observable<Room[]>;
  availabilityConfirmed$: Observable<boolean>;

  constructor(
    private fb: FormBuilder,
    private roomService: RoomService,
    private bookingService: BookingService
  ) {
    this.buildForm();
  }

  ngOnInit() {
    this.setupValidationObservables();
    this.setupCalculatedValues();
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }

  private buildForm(): void {
    const tomorrow = new Date();
    tomorrow.setDate(tomorrow.getDate() + 1);
    
    const dayAfter = new Date();
    dayAfter.setDate(dayAfter.getDate() + 2);

    this.bookingForm = this.fb.group({
      checkInDate: [tomorrow.toISOString().split('T')[0], Validators.required],
      checkOutDate: [dayAfter.toISOString().split('T')[0], Validators.required],
      guests: [1, [Validators.required, Validators.min(1), Validators.max(10)]],
      roomType: ['', Validators.required]
    });
  }

  private setupValidationObservables(): void {
    // Individual field validation
    this.checkInInvalid$ = this.createFieldInvalidObservable('checkInDate');
    this.checkOutInvalid$ = this.createFieldInvalidObservable('checkOutDate');
    this.guestsInvalid$ = this.createFieldInvalidObservable('guests');
    this.roomTypeInvalid$ = this.createFieldInvalidObservable('roomType');

    // Cross-field validation
    const dateRange$ = combineLatest([
      this.bookingForm.get('checkInDate')!.valueChanges.pipe(startWith(this.bookingForm.get('checkInDate')!.value)),
      this.bookingForm.get('checkOutDate')!.valueChanges.pipe(startWith(this.bookingForm.get('checkOutDate')!.value))
    ]);

    this.dateRangeError$ = dateRange$.pipe(
      map(([checkIn, checkOut]) => {
        if (!checkIn || !checkOut) return '';
        
        const checkInDate = new Date(checkIn);
        const checkOutDate = new Date(checkOut);
        const today = new Date();
        today.setHours(0, 0, 0, 0);

        if (checkInDate < today) {
          return 'Check-in date cannot be in the past';
        }

        if (checkOutDate <= checkInDate) {
          return 'Check-out date must be after check-in date';
        }

        const daysDiff = (checkOutDate.getTime() - checkInDate.getTime()) / (1000 * 3600 * 24);
        if (daysDiff > 30) {
          return 'Maximum stay is 30 nights';
        }

        return '';
      })
    );

    // Room capacity validation
    this.capacityError$ = combineLatest([
      this.bookingForm.get('guests')!.valueChanges.pipe(startWith(this.bookingForm.get('guests')!.value)),
      this.bookingForm.get('roomType')!.valueChanges.pipe(startWith(this.bookingForm.get('roomType')!.value)),
      this.availableRooms$
    ]).pipe(
      map(([guests, roomTypeId, rooms]) => {
        if (!guests || !roomTypeId || !rooms) return '';
        
        const selectedRoom = rooms.find(room => room.id === roomTypeId);
        if (selectedRoom && guests > selectedRoom.maxCapacity) {
          return `Selected room can accommodate maximum ${selectedRoom.maxCapacity} guests`;
        }
        
        return '';
      })
    );

    // Availability validation
    this.availabilityError$ = combineLatest([
      dateRange$,
      this.bookingForm.get('roomType')!.valueChanges.pipe(startWith(this.bookingForm.get('roomType')!.value))
    ]).pipe(
      debounceTime(500),
      switchMap(([[checkIn, checkOut], roomTypeId]) => {
        if (!checkIn || !checkOut || !roomTypeId) return of('');
        
        return this.bookingService.checkAvailability(roomTypeId, checkIn, checkOut).pipe(
          map(available => available ? '' : 'Selected room is not available for chosen dates'),
          catchError(() => of('Unable to check availability'))
        );
      })
    );
  }

  private setupCalculatedValues(): void {
    // Available rooms based on dates and guests
    this.availableRooms$ = combineLatest([
      this.bookingForm.get('checkInDate')!.valueChanges.pipe(startWith(this.bookingForm.get('checkInDate')!.value)),
      this.bookingForm.get('checkOutDate')!.valueChanges.pipe(startWith(this.bookingForm.get('checkOutDate')!.value)),
      this.bookingForm.get('guests')!.valueChanges.pipe(startWith(this.bookingForm.get('guests')!.value))
    ]).pipe(
      debounceTime(300),
      switchMap(([checkIn, checkOut, guests]) => {
        if (!checkIn || !checkOut || !guests) return of([]);
        
        return this.roomService.getAvailableRooms(checkIn, checkOut, guests);
      }),
      shareReplay(1)
    );

    // Night count calculation
    this.nightCount$ = combineLatest([
      this.bookingForm.get('checkInDate')!.valueChanges.pipe(startWith(this.bookingForm.get('checkInDate')!.value)),
      this.bookingForm.get('checkOutDate')!.valueChanges.pipe(startWith(this.bookingForm.get('checkOutDate')!.value))
    ]).pipe(
      map(([checkIn, checkOut]) => {
        if (!checkIn || !checkOut) return 0;
        
        const checkInDate = new Date(checkIn);
        const checkOutDate = new Date(checkOut);
        const timeDiff = checkOutDate.getTime() - checkInDate.getTime();
        
        return Math.max(0, Math.ceil(timeDiff / (1000 * 3600 * 24)));
      })
    );

    // Total price calculation
    this.totalPrice$ = combineLatest([
      this.nightCount$,
      this.bookingForm.get('roomType')!.valueChanges.pipe(startWith(this.bookingForm.get('roomType')!.value)),
      this.availableRooms$
    ]).pipe(
      map(([nights, roomTypeId, rooms]) => {
        if (!nights || !roomTypeId || !rooms) return 0;
        
        const selectedRoom = rooms.find(room => room.id === roomTypeId);
        return selectedRoom ? nights * selectedRoom.price : 0;
      })
    );

    // Availability confirmation
    this.availabilityConfirmed$ = this.availabilityError$.pipe(
      map(error => error === '')
    );
  }

  private createFieldInvalidObservable(fieldName: string): Observable<boolean> {
    const control = this.bookingForm.get(fieldName)!;
    
    return combineLatest([
      control.statusChanges.pipe(startWith(control.status)),
      control.valueChanges.pipe(startWith(control.value))
    ]).pipe(
      map(() => control.invalid && control.touched)
    );
  }

  onSubmit(): void {
    if (this.bookingForm.valid) {
      const bookingData = {
        ...this.bookingForm.value,
        totalPrice: this.totalPrice$.pipe(take(1)),
        nights: this.nightCount$.pipe(take(1))
      };

      this.bookingService.createBooking(bookingData).pipe(
        takeUntil(this.destroy$)
      ).subscribe({
        next: (booking) => {
          console.log('Booking created:', booking);
          // Handle success
        },
        error: (error) => {
          console.error('Booking failed:', error);
          // Handle error
        }
      });
    }
  }
}

interface Room {
  id: string;
  name: string;
  price: number;
  maxCapacity: number;
}
```

### Form Array Management with RxJS

**Dynamic Form Arrays:**
```typescript
@Component({
  selector: 'app-invoice-form',
  template: `
    <form [formGroup]="invoiceForm">
      <div class="invoice-header">
        <div class="form-field">
          <label>Client Name</label>
          <input formControlName="clientName" [class.error]="clientNameInvalid$ | async">
        </div>
        
        <div class="form-field">
          <label>Invoice Date</label>
          <input type="date" formControlName="invoiceDate">
        </div>
      </div>

      <div class="invoice-items">
        <h3>Invoice Items</h3>
        <div formArrayName="items">
          <div 
            *ngFor="let item of itemsArray.controls; let i = index" 
            [formGroupName]="i" 
            class="invoice-item">
            
            <div class="item-row">
              <div class="form-field">
                <label>Description</label>
                <input formControlName="description">
              </div>
              
              <div class="form-field">
                <label>Quantity</label>
                <input type="number" formControlName="quantity">
              </div>
              
              <div class="form-field">
                <label>Unit Price</label>
                <input type="number" step="0.01" formControlName="unitPrice">
              </div>
              
              <div class="form-field">
                <label>Total</label>
                <span class="calculated-value">
                  {{ getItemTotal(i) | async | currency }}
                </span>
              </div>
              
              <button type="button" (click)="removeItem(i)" class="remove-btn">
                Remove
              </button>
            </div>
          </div>
        </div>

        <button type="button" (click)="addItem()" class="add-btn">
          Add Item
        </button>
      </div>

      <div class="invoice-summary">
        <div class="summary-row">
          <span>Subtotal:</span>
          <span>{{ subtotal$ | async | currency }}</span>
        </div>
        
        <div class="summary-row">
          <span>Tax ({{ taxRate }}%):</span>
          <span>{{ tax$ | async | currency }}</span>
        </div>
        
        <div class="summary-row total">
          <span>Total:</span>
          <span>{{ total$ | async | currency }}</span>
        </div>
      </div>

      <button 
        type="submit" 
        [disabled]="!invoiceForm.valid || (submitting$ | async)"
        (click)="onSubmit()">
        {{ (submitting$ | async) ? 'Creating Invoice...' : 'Create Invoice' }}
      </button>
    </form>
  `
})
export class InvoiceFormComponent implements OnInit, OnDestroy {
  private destroy$ = new Subject<void>();
  
  invoiceForm: FormGroup;
  taxRate = 10; // 10% tax
  submitting$ = new BehaviorSubject(false);

  // Form observables
  clientNameInvalid$: Observable<boolean>;
  subtotal$: Observable<number>;
  tax$: Observable<number>;
  total$: Observable<number>;

  constructor(
    private fb: FormBuilder,
    private invoiceService: InvoiceService
  ) {
    this.buildForm();
  }

  ngOnInit() {
    this.setupFormObservables();
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }

  get itemsArray(): FormArray {
    return this.invoiceForm.get('items') as FormArray;
  }

  private buildForm(): void {
    this.invoiceForm = this.fb.group({
      clientName: ['', Validators.required],
      invoiceDate: [new Date().toISOString().split('T')[0], Validators.required],
      items: this.fb.array([this.createItemGroup()])
    });
  }

  private createItemGroup(): FormGroup {
    return this.fb.group({
      description: ['', Validators.required],
      quantity: [1, [Validators.required, Validators.min(1)]],
      unitPrice: [0, [Validators.required, Validators.min(0.01)]]
    });
  }

  private setupFormObservables(): void {
    const clientNameControl = this.invoiceForm.get('clientName')!;
    this.clientNameInvalid$ = combineLatest([
      clientNameControl.statusChanges.pipe(startWith(clientNameControl.status)),
      clientNameControl.valueChanges.pipe(startWith(clientNameControl.value))
    ]).pipe(
      map(() => clientNameControl.invalid && clientNameControl.touched)
    );

    // Calculate subtotal from all items
    this.subtotal$ = this.itemsArray.valueChanges.pipe(
      startWith(this.itemsArray.value),
      map(items => 
        items.reduce((sum: number, item: any) => 
          sum + (item.quantity * item.unitPrice), 0
        )
      )
    );

    // Calculate tax
    this.tax$ = this.subtotal$.pipe(
      map(subtotal => subtotal * this.taxRate / 100)
    );

    // Calculate total
    this.total$ = combineLatest([this.subtotal$, this.tax$]).pipe(
      map(([subtotal, tax]) => subtotal + tax)
    );
  }

  getItemTotal(index: number): Observable<number> {
    const itemGroup = this.itemsArray.at(index);
    
    return combineLatest([
      itemGroup.get('quantity')!.valueChanges.pipe(startWith(itemGroup.get('quantity')!.value)),
      itemGroup.get('unitPrice')!.valueChanges.pipe(startWith(itemGroup.get('unitPrice')!.value))
    ]).pipe(
      map(([quantity, unitPrice]) => (quantity || 0) * (unitPrice || 0))
    );
  }

  addItem(): void {
    this.itemsArray.push(this.createItemGroup());
  }

  removeItem(index: number): void {
    if (this.itemsArray.length > 1) {
      this.itemsArray.removeAt(index);
    }
  }

  onSubmit(): void {
    if (this.invoiceForm.valid) {
      this.submitting$.next(true);

      combineLatest([
        of(this.invoiceForm.value),
        this.subtotal$,
        this.tax$,
        this.total$
      ]).pipe(
        take(1),
        switchMap(([formValue, subtotal, tax, total]) => {
          const invoiceData = {
            ...formValue,
            subtotal,
            tax,
            total,
            taxRate: this.taxRate
          };
          
          return this.invoiceService.createInvoice(invoiceData);
        }),
        takeUntil(this.destroy$),
        finalize(() => this.submitting$.next(false))
      ).subscribe({
        next: (invoice) => {
          console.log('Invoice created:', invoice);
          this.invoiceForm.reset();
          this.itemsArray.clear();
          this.addItem(); // Add one default item
        },
        error: (error) => {
          console.error('Failed to create invoice:', error);
        }
      });
    }
  }
}
```

## Advanced Reactive Form Patterns

### 1. Real-time Form Synchronization

```typescript
@Injectable({
  providedIn: 'root'
})
export class FormSyncService {
  private formUpdates$ = new Subject<{ formId: string; data: any }>();

  constructor(private websocketService: WebSocketService) {
    // Send form updates to server
    this.formUpdates$.pipe(
      debounceTime(1000),
      distinctUntilChanged((prev, curr) => 
        prev.formId === curr.formId && JSON.stringify(prev.data) === JSON.stringify(curr.data)
      )
    ).subscribe(update => {
      this.websocketService.send('form_update', update);
    });

    // Receive form updates from server
    this.websocketService.on('form_update').subscribe(update => {
      this.handleRemoteFormUpdate(update);
    });
  }

  syncForm(formId: string, form: FormGroup): Observable<any> {
    // Send form changes to other users
    form.valueChanges.pipe(
      debounceTime(500)
    ).subscribe(data => {
      this.formUpdates$.next({ formId, data });
    });

    // Return observable for receiving updates
    return this.websocketService.on('form_update').pipe(
      filter(update => update.formId === formId),
      map(update => update.data)
    );
  }

  private handleRemoteFormUpdate(update: any): void {
    // Handle incoming form updates from other users
    console.log('Received form update:', update);
  }
}
```

### 2. Form Wizard with Step Validation

```typescript
@Component({
  selector: 'app-form-wizard',
  template: `
    <div class="wizard-container">
      <div class="wizard-steps">
        <div 
          *ngFor="let step of steps; let i = index"
          class="step"
          [class.active]="currentStep === i"
          [class.completed]="isStepCompleted(i) | async">
          {{ step.title }}
        </div>
      </div>

      <form [formGroup]="wizardForm">
        <div [ngSwitch]="currentStep">
          <!-- Step 1: Personal Info -->
          <div *ngSwitchCase="0" formGroupName="personal">
            <h2>Personal Information</h2>
            <div class="form-field">
              <label>First Name</label>
              <input formControlName="firstName">
            </div>
            <div class="form-field">
              <label>Last Name</label>
              <input formControlName="lastName">
            </div>
            <div class="form-field">
              <label>Email</label>
              <input formControlName="email">
            </div>
          </div>

          <!-- Step 2: Address -->
          <div *ngSwitchCase="1" formGroupName="address">
            <h2>Address Information</h2>
            <div class="form-field">
              <label>Street</label>
              <input formControlName="street">
            </div>
            <div class="form-field">
              <label>City</label>
              <input formControlName="city">
            </div>
            <div class="form-field">
              <label>Postal Code</label>
              <input formControlName="postalCode">
            </div>
          </div>

          <!-- Step 3: Preferences -->
          <div *ngSwitchCase="2" formGroupName="preferences">
            <h2>Preferences</h2>
            <div class="form-field">
              <label>Newsletter</label>
              <input type="checkbox" formControlName="newsletter">
            </div>
            <div class="form-field">
              <label>Notifications</label>
              <input type="checkbox" formControlName="notifications">
            </div>
          </div>
        </div>

        <div class="wizard-navigation">
          <button 
            type="button" 
            (click)="previousStep()" 
            [disabled]="currentStep === 0">
            Previous
          </button>
          
          <button 
            type="button" 
            (click)="nextStep()" 
            [disabled]="!(canProceed$ | async)"
            *ngIf="currentStep < steps.length - 1">
            Next
          </button>
          
          <button 
            type="submit" 
            (click)="onSubmit()"
            [disabled]="!wizardForm.valid || (submitting$ | async)"
            *ngIf="currentStep === steps.length - 1">
            {{ (submitting$ | async) ? 'Submitting...' : 'Submit' }}
          </button>
        </div>
      </form>
    </div>
  `
})
export class FormWizardComponent implements OnInit {
  currentStep = 0;
  submitting$ = new BehaviorSubject(false);
  
  steps = [
    { title: 'Personal', group: 'personal' },
    { title: 'Address', group: 'address' },
    { title: 'Preferences', group: 'preferences' }
  ];

  wizardForm: FormGroup;
  canProceed$: Observable<boolean>;

  constructor(private fb: FormBuilder) {
    this.buildForm();
  }

  ngOnInit() {
    this.setupStepValidation();
  }

  private buildForm(): void {
    this.wizardForm = this.fb.group({
      personal: this.fb.group({
        firstName: ['', Validators.required],
        lastName: ['', Validators.required],
        email: ['', [Validators.required, Validators.email]]
      }),
      address: this.fb.group({
        street: ['', Validators.required],
        city: ['', Validators.required],
        postalCode: ['', Validators.required]
      }),
      preferences: this.fb.group({
        newsletter: [false],
        notifications: [true]
      })
    });
  }

  private setupStepValidation(): void {
    this.canProceed$ = new Observable(subscriber => {
      const updateCanProceed = () => {
        const currentStepGroup = this.getCurrentStepGroup();
        subscriber.next(currentStepGroup ? currentStepGroup.valid : false);
      };

      // Initial check
      updateCanProceed();

      // Listen for form changes
      const subscription = this.wizardForm.statusChanges.subscribe(() => {
        updateCanProceed();
      });

      return () => subscription.unsubscribe();
    });
  }

  private getCurrentStepGroup(): AbstractControl | null {
    const stepConfig = this.steps[this.currentStep];
    return this.wizardForm.get(stepConfig.group);
  }

  isStepCompleted(stepIndex: number): Observable<boolean> {
    const stepConfig = this.steps[stepIndex];
    const stepGroup = this.wizardForm.get(stepConfig.group);
    
    if (!stepGroup) return of(false);

    return stepGroup.statusChanges.pipe(
      startWith(stepGroup.status),
      map(status => status === 'VALID')
    );
  }

  nextStep(): void {
    const currentStepGroup = this.getCurrentStepGroup();
    if (currentStepGroup && currentStepGroup.valid && this.currentStep < this.steps.length - 1) {
      this.currentStep++;
    }
  }

  previousStep(): void {
    if (this.currentStep > 0) {
      this.currentStep--;
    }
  }

  onSubmit(): void {
    if (this.wizardForm.valid) {
      this.submitting$.next(true);
      
      // Flatten the form data
      const formData = {
        ...this.wizardForm.value.personal,
        ...this.wizardForm.value.address,
        ...this.wizardForm.value.preferences
      };

      // Submit the data
      console.log('Submitting wizard data:', formData);
      
      // Simulate API call
      setTimeout(() => {
        this.submitting$.next(false);
        console.log('Wizard submission completed');
      }, 2000);
    }
  }
}
```

## Best Practices

### 1. Performance Optimization
```typescript
// ✅ Use OnPush change detection with reactive forms
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class OptimizedFormComponent {
  // Reactive forms work perfectly with OnPush
}

// ✅ Debounce form value changes
this.form.valueChanges.pipe(
  debounceTime(300),
  distinctUntilChanged()
).subscribe(value => {
  // Handle form changes
});
```

### 2. Error Handling
```typescript
// ✅ Centralized error message handling
getErrorMessage(control: AbstractControl): string {
  if (control.errors?.['required']) return 'This field is required';
  if (control.errors?.['email']) return 'Please enter a valid email';
  if (control.errors?.['minlength']) {
    const requiredLength = control.errors['minlength'].requiredLength;
    return `Minimum length is ${requiredLength} characters`;
  }
  return '';
}
```

### 3. Memory Management
```typescript
// ✅ Always clean up subscriptions
ngOnDestroy() {
  this.destroy$.next();
  this.destroy$.complete();
}

// ✅ Use takeUntil for automatic cleanup
this.form.valueChanges.pipe(
  takeUntil(this.destroy$)
).subscribe(value => {
  // Handle value changes
});
```

## Summary

Reactive Forms with RxJS provide powerful capabilities for building sophisticated forms:

- **Real-time Validation**: Immediate feedback with observable-based validation
- **Cross-field Validation**: Complex validation rules across multiple fields
- **Dynamic Forms**: Forms that adapt based on user input or external data
- **Form Arrays**: Managing dynamic lists of form controls
- **Advanced Patterns**: Wizards, real-time sync, conditional logic

Key principles:
- Use observables for form state management
- Implement proper error handling and user feedback
- Leverage RxJS operators for complex validation logic
- Optimize performance with proper change detection strategies
- Always clean up subscriptions to prevent memory leaks
