# Angular typed reactive form

Adapted from the private standalone project form component. Large inline styles and most presentation markup are intentionally omitted.

```ts
export interface ProjectFormValue {
  name: string;
  description: string;
  dueDate: string;
  status: 'Planning' | 'In progress' | 'At risk' | 'Completed';
}

@Component({
  selector: 'app-project-form',
  standalone: true,
  imports: [ReactiveFormsModule],
  template: `
    <form [formGroup]="form" (ngSubmit)="submit()" novalidate>
      <label for="project-name">Project name</label>
      <input
        id="project-name"
        formControlName="name"
        [attr.aria-invalid]="hasError('name')"
      />
      @if (hasError('name')) {
        <small>Enter a project name.</small>
      }

      <label for="project-description">Description</label>
      <textarea
        id="project-description"
        formControlName="description"
        [attr.aria-invalid]="hasError('description')"
      ></textarea>
      @if (hasError('description')) {
        <small>Add a short project description.</small>
      }

      <button type="submit">Create project</button>
    </form>
  `,
})
export class ProjectFormComponent {
  private readonly formBuilder = inject(FormBuilder);
  @Output() readonly created = new EventEmitter<ProjectFormValue>();

  readonly form = this.formBuilder.nonNullable.group({
    name: ['', [Validators.required, Validators.minLength(2)]],
    description: ['', [Validators.required, Validators.minLength(10)]],
    dueDate: ['', Validators.required],
    status: ['Planning' as ProjectFormValue['status'], Validators.required],
  });

  hasError(name: keyof ProjectFormValue): boolean {
    const control = this.form.controls[name];
    return control.invalid && (control.dirty || control.touched);
  }

  submit(): void {
    this.form.markAllAsTouched();
    if (this.form.invalid) return;
    this.created.emit(this.form.getRawValue());
  }
}
```

## What this demonstrates

- a standalone Angular component with a non-nullable typed form;
- validation at the form boundary before emitting a typed value;
- visible validation messages within their associated labels;
- accessible invalid-state signaling without reproducing the full modal UI.
