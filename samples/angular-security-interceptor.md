# Angular security interceptor

Adapted from the private functional authentication interceptor. The excerpt uses a relative API boundary and contains no credentials or deployment configuration.

```ts
const mutationMethods = new Set(['POST', 'PUT', 'PATCH', 'DELETE']);

export const authInterceptor: HttpInterceptorFn = (request, next) => {
  const session = inject(AuthSessionStore);
  const protectedState = inject(ProtectedStateService);
  const router = inject(Router);

  if (!request.url.startsWith('/api')) return next(request);

  const csrfToken = readCookie('XSRF-TOKEN');
  const securedRequest = request.clone({
    withCredentials: true,
    setHeaders:
      mutationMethods.has(request.method) && csrfToken
        ? { 'X-XSRF-TOKEN': csrfToken }
        : {},
  });

  return next(securedRequest).pipe(
    catchError((error: unknown) => {
      const sessionExpired =
        error instanceof HttpErrorResponse &&
        error.status === 401 &&
        !isAuthenticationRequest(request.url);

      if (sessionExpired) {
        const returnUrl = safeReturnUrl(router.url);
        protectedState.clear();
        session.anonymous(true);
        void router.navigate(['/login'], {
          queryParams: { reason: 'expired', returnUrl },
          replaceUrl: true,
        });
      }
      return throwError(() => error);
    }),
  );
};

function readCookie(name: string): string | null {
  if (typeof document === 'undefined') return null;
  const prefix = `${encodeURIComponent(name)}=`;
  const part = document.cookie
    .split(';')
    .map((value) => value.trim())
    .find((value) => value.startsWith(prefix));
  return part ? decodeURIComponent(part.slice(prefix.length)) : null;
}

function isAuthenticationRequest(url: string): boolean {
  return ['/api/auth/login', '/api/auth/session', '/api/auth/antiforgery']
    .some((path) => url.includes(path));
}

export function safeReturnUrl(value?: string | null): string {
  return value?.startsWith('/') && !value.startsWith('//') && !value.startsWith('/login')
    ? value
    : '/dashboard';
}
```

## What this demonstrates

- credentials are scoped to the application API boundary;
- antiforgery headers are added only to state-changing requests;
- expired protected sessions clear client-owned protected state;
- return URLs are restricted to safe local application paths.
