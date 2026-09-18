# Angular signal-based project state

Adapted from the private `ProjectService`. Response mapping and state changes stay centralized, and duplicate detail requests are prevented without publishing the complete data service.

```ts
interface ProjectResponse {
  id: number;
  name: string;
  description: string;
  status: 'Planning' | 'InProgress' | 'AtRisk' | 'Completed';
  dueDateUtc: string;
  progress: number;
}

interface Project {
  id: number;
  name: string;
  description: string;
  status: 'Planning' | 'In progress' | 'At risk' | 'Completed';
  dueDate: string;
  progress: number;
}

@Injectable({ providedIn: 'root' })
export class ProjectService {
  private readonly http = inject(HttpClient);
  private readonly state = signal<Project[]>([]);
  private readonly pendingIds = new Set<number>();

  readonly projects = this.state.asReadonly();
  readonly loading = signal(false);

  loadProject(id: number, force = false): void {
    const cached = this.state().some((project) => project.id === id);
    if (this.pendingIds.has(id) || (!force && cached)) return;

    this.pendingIds.add(id);
    this.loading.set(true);
    this.http.get<ProjectResponse>(`/api/projects/${id}`).pipe(
      map((response) => this.mapProject(response)),
      finalize(() => {
        this.pendingIds.delete(id);
        this.loading.set(this.pendingIds.size > 0);
      }),
    ).subscribe((project) => this.upsert(project));
  }

  private upsert(project: Project): void {
    this.state.update((projects) =>
      projects.some((item) => item.id === project.id)
        ? projects.map((item) => item.id === project.id ? project : item)
        : [...projects, project].sort((a, b) => a.id - b.id),
    );
  }

  private mapProject(response: ProjectResponse): Project {
    return {
      ...response,
      dueDate: response.dueDateUtc.slice(0, 10),
      status: response.status === 'InProgress'
        ? 'In progress'
        : response.status === 'AtRisk' ? 'At risk' : response.status,
    };
  }
}
```

## What this demonstrates

- explicit API and UI models at the transport boundary;
- read-only signal exposure to consuming components;
- per-project request de-duplication and cache-aware loading;
- immutable replacement or sorted insertion without duplicate records.
