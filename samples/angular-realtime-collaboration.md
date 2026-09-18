# Angular real-time collaboration

Adapted from the private `TaskCommentService`. The excerpt keeps the typed state and connection lifecycle while omitting HTTP mutations, notification implementation, and unrelated helpers.

```ts
export interface TaskComment {
  id: number;
  taskId: number;
  content: string;
  createdAtUtc: string;
  editedAtUtc: string | null;
}

type ConnectionState =
  | 'disconnected'
  | 'connecting'
  | 'connected'
  | 'reconnecting'
  | 'unavailable';

@Injectable({ providedIn: 'root' })
export class TaskCommentService {
  private readonly commentsByTask = signal<Record<number, TaskComment[]>>({});
  private connection?: HubConnection;
  private activeTaskId: number | null = null;

  readonly connectionState = signal<ConnectionState>('disconnected');
  readonly activeComments = computed(() =>
    this.activeTaskId === null
      ? []
      : (this.commentsByTask()[this.activeTaskId] ?? []),
  );

  async connect(taskId: number): Promise<void> {
    this.activeTaskId = taskId;
    this.connection ??= this.buildConnection();

    try {
      if (this.connection.state === HubConnectionState.Disconnected) {
        this.connectionState.set('connecting');
        await this.connection.start();
      }
      await this.connection.invoke('JoinTask', taskId);
      this.connectionState.set('connected');
    } catch {
      this.connectionState.set('unavailable');
    }
  }

  private buildConnection(): HubConnection {
    const connection = this.connectionFactory();
    connection.on('CommentCreated', (comment: TaskComment) => this.upsert(comment));
    connection.on('CommentUpdated', (comment: TaskComment) => this.upsert(comment));
    connection.onreconnecting(() => this.connectionState.set('reconnecting'));
    connection.onreconnected(async () => {
      if (this.activeTaskId !== null) {
        await connection.invoke('JoinTask', this.activeTaskId);
      }
      this.connectionState.set('connected');
    });
    connection.onclose(() => this.connectionState.set('unavailable'));
    return connection;
  }
}
```

## What this demonstrates

- typed collaboration state exposed through Angular signals;
- a computed view for the currently active task;
- explicit connecting, reconnecting, connected, and unavailable states;
- authenticated room membership and automatic task-room rejoin after transport recovery.
