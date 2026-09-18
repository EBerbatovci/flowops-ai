# Angular collaboration test

Adapted from the private task-comment collaboration tests. The mock keeps the important SignalR lifecycle while synthetic identifiers and content avoid publishing product data.

```ts
describe('TaskCommentService collaboration', () => {
  let service: TaskCommentService;
  let http: HttpTestingController;
  let hubState = HubConnectionState.Disconnected;
  let reconnecting: () => void;
  let reconnected: () => Promise<void>;
  const invocations: Array<[string, number]> = [];

  const hub = {
    get state() { return hubState; },
    start: async () => { hubState = HubConnectionState.Connected; },
    stop: async () => { hubState = HubConnectionState.Disconnected; },
    invoke: async (method: string, taskId: number) => {
      invocations.push([method, taskId]);
    },
    on: () => undefined,
    onreconnecting: (callback: () => void) => { reconnecting = callback; },
    onreconnected: (callback: () => Promise<void>) => { reconnected = callback; },
    onclose: () => undefined,
  } as unknown as HubConnection;

  beforeEach(() => {
    hubState = HubConnectionState.Disconnected;
    invocations.length = 0;
    TestBed.configureTestingModule({
      providers: [
        provideHttpClient(),
        provideHttpClientTesting(),
        { provide: TASK_HUB_CONNECTION, useValue: () => hub },
      ],
    });
    service = TestBed.inject(TaskCommentService);
    http = TestBed.inject(HttpTestingController);
  });

  afterEach(() => http.verify());

  it('tracks reconnect state and rejoins the active task room', async () => {
    await service.connect(42);
    expect(service.connectionState()).toBe('connected');
    expect(invocations).toContainEqual(['JoinTask', 42]);

    reconnecting();
    expect(service.connectionState()).toBe('reconnecting');

    await reconnected();
    expect(service.connectionState()).toBe('connected');
    expect(invocations.filter(([method]) => method === 'JoinTask')).toHaveLength(2);
  });
});
```

## What this demonstrates

- dependency injection through Angular TestBed;
- the HTTP testing controller as a guard against unexpected requests;
- a deterministic SignalR mock with observable lifecycle callbacks;
- verification that reconnection restores the active collaboration room.
