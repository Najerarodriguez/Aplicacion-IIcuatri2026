# Guía aplicada para crear una suite de pruebas unitarias con 50% de cobertura

## Resultado esperado

Al terminar estos pasos tendrás:

- Pruebas de renderizado de componentes.
- Pruebas de cambios de estado por acciones del usuario.
- Pruebas de hooks personalizados.
- Pruebas de funciones utilitarias y transformación de datos.
- Pruebas de manejo de errores.
- Medición de cobertura con meta mínima de 50%.

## Paso 1: Preparar dependencias

Ejecuta:

```bash
npm install
```

Instala cobertura para Vitest:

```bash
npm install -D @vitest/coverage-v8
```

## Paso 2: Configurar cobertura en Vite

Abre:

```text
vite.config.js
```

Déjalo así:

```js
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    globals: true,
    setupFiles: './src/tests/setup.js',
    css: true,
    coverage: {
      provider: 'v8',
      reporter: ['text', 'html'],
      thresholds: {
        lines: 50,
        functions: 50,
        branches: 50,
        statements: 50
      }
    }
  }
});
```

## Paso 3: Agregar script de cobertura

Abre:

```text
package.json
```

En la sección `scripts`, agrega:

```json
"test:coverage": "vitest run --coverage"
```

La sección puede quedar así:

```json
"scripts": {
  "dev": "vite",
  "build": "vite build",
  "preview": "vite preview",
  "lint": "eslint .",
  "test": "vitest run",
  "test:watch": "vitest",
  "test:coverage": "vitest run --coverage",
  "test:e2e": "playwright test"
}
```

## Paso 4: Crear pruebas de componentes

Crea este archivo:

```text
src/tests/componentes-aplicados.test.jsx
```

Pega este contenido:

```jsx
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { MemoryRouter } from 'react-router-dom';
import { describe, expect, it, vi } from 'vitest';
import { AuthFormFields } from '../components/AuthFormFields';
import { AuthLayout } from '../components/AuthLayout';
import { ErrorBanner } from '../components/ErrorBanner';
import { LoadingSpinner } from '../components/LoadingSpinner';
import { StatsCards } from '../components/StatsCards';
import { TaskForm } from '../components/TaskForm';
import { TaskList } from '../components/TaskList';

describe('Componentes aplicados', () => {
  it('renderiza ErrorBanner con mensaje visible', () => {
    render(<ErrorBanner message="Error controlado" />);

    expect(screen.getByRole('alert')).toBeInTheDocument();
    expect(screen.getByText('Error controlado')).toBeInTheDocument();
  });

  it('renderiza LoadingSpinner con etiqueta personalizada', () => {
    render(<LoadingSpinner label="Cargando datos" />);

    expect(screen.getByText('Cargando datos')).toBeInTheDocument();
    expect(screen.getByText('Cargando datos').parentElement).toHaveAttribute('aria-busy', 'true');
  });

  it('renderiza AuthLayout con titulo, contenido y enlace', () => {
    render(
      <MemoryRouter>
        <AuthLayout
          footerLabel="Ir al login"
          footerLink="/login"
          footerText="Ya tienes cuenta"
          subtitle="Subtitulo visible"
          title="Titulo visible"
        >
          <button type="button">Accion interna</button>
        </AuthLayout>
      </MemoryRouter>
    );

    expect(screen.getByRole('heading', { name: 'Titulo visible' })).toBeInTheDocument();
    expect(screen.getByText('Subtitulo visible')).toBeInTheDocument();
    expect(screen.getByRole('button', { name: 'Accion interna' })).toBeInTheDocument();
    expect(screen.getByRole('link', { name: 'Ir al login' })).toHaveAttribute('href', '/login');
  });

  it('renderiza AuthFormFields en modo login', () => {
    render(
      <AuthFormFields
        errors={{}}
        mode="login"
        onChange={vi.fn()}
        values={{ email: 'persona@test.com', password: '123' }}
      />
    );

    expect(screen.getByLabelText('Correo')).toHaveValue('persona@test.com');
    expect(screen.getByLabelText('Contraseña')).toHaveValue('123');
    expect(screen.queryByLabelText('Nombre')).not.toBeInTheDocument();
  });

  it('renderiza AuthFormFields en modo registro con errores', () => {
    render(
      <AuthFormFields
        errors={{
          displayName: 'Nombre requerido',
          confirmPassword: 'Confirmacion requerida'
        }}
        mode="register"
        onChange={vi.fn()}
        values={{
          displayName: '',
          email: '',
          password: '',
          confirmPassword: ''
        }}
      />
    );

    expect(screen.getByLabelText('Nombre')).toBeInTheDocument();
    expect(screen.getByLabelText('Confirmar contraseña')).toBeInTheDocument();
    expect(screen.getByText('Nombre requerido')).toBeInTheDocument();
    expect(screen.getByText('Confirmacion requerida')).toBeInTheDocument();
  });

  it('renderiza StatsCards con diferentes props', () => {
    render(
      <StatsCards
        stats={{
          total: 4,
          pending: 2,
          'in-progress': 1,
          completed: 1
        }}
      />
    );

    expect(screen.getByText('Total')).toBeInTheDocument();
    expect(screen.getByText('4')).toBeInTheDocument();
    expect(screen.getByText('Pendientes')).toBeInTheDocument();
    expect(screen.getByText('2')).toBeInTheDocument();
    expect(screen.getByText('En progreso')).toBeInTheDocument();
    expect(screen.getAllByText('1')).toHaveLength(2);
    expect(screen.getByText('Completadas')).toBeInTheDocument();
  });

  it('renderiza TaskList en estado de carga', () => {
    render(<TaskList loading onDelete={vi.fn()} onEdit={vi.fn()} tasks={[]} />);

    expect(screen.getByText('Cargando tareas...')).toBeInTheDocument();
  });

  it('renderiza TaskList cuando la lista esta vacia', () => {
    render(<TaskList loading={false} onDelete={vi.fn()} onEdit={vi.fn()} tasks={[]} />);

    expect(screen.getByText('0 registros')).toBeInTheDocument();
    expect(screen.getByText('Aun no hay tareas. Crea la primera desde el formulario.')).toBeInTheDocument();
  });

  it('ejecuta editar y eliminar desde TaskList', async () => {
    const user = userEvent.setup();
    const onEdit = vi.fn();
    const onDelete = vi.fn();
    const task = {
      id: 'task-1',
      title: '<strong>Tarea visible</strong>',
      description: '<em>Descripcion visible</em>',
      status: 'pending',
      updatedAt: Date.now()
    };

    render(<TaskList loading={false} onDelete={onDelete} onEdit={onEdit} tasks={[task]} />);

    expect(screen.getByText('1 registros')).toBeInTheDocument();
    expect(screen.getByText('Tarea visible')).toBeInTheDocument();

    await user.click(screen.getByRole('button', { name: 'Editar' }));
    await user.click(screen.getByRole('button', { name: 'Eliminar' }));

    expect(onEdit).toHaveBeenCalledWith(task);
    expect(onDelete).toHaveBeenCalledWith('task-1');
  });

  it('cambia estado interno de TaskForm y envia una tarea nueva', async () => {
    const user = userEvent.setup();
    const onSubmit = vi.fn().mockResolvedValue(undefined);

    render(<TaskForm loading={false} onCancel={vi.fn()} onSubmit={onSubmit} />);

    await user.type(screen.getByLabelText('Titulo'), 'Nueva tarea');
    await user.type(screen.getByLabelText(/Descripcion/), 'Descripcion nueva');
    await user.selectOptions(screen.getByLabelText('Estado'), 'completed');
    await user.click(screen.getByRole('button', { name: 'Crear tarea' }));

    await waitFor(() => {
      expect(onSubmit).toHaveBeenCalledWith({
        title: 'Nueva tarea',
        description: 'Descripcion nueva',
        status: 'completed'
      });
    });
  });

  it('renderiza TaskForm en modo edicion y permite cancelar', async () => {
    const user = userEvent.setup();
    const onCancel = vi.fn();

    render(
      <TaskForm
        initialTask={{
          id: 'task-2',
          title: 'Titulo inicial',
          description: 'Descripcion inicial',
          status: 'in-progress'
        }}
        loading={false}
        onCancel={onCancel}
        onSubmit={vi.fn()}
      />
    );

    expect(screen.getByRole('heading', { name: 'Editar tarea' })).toBeInTheDocument();
    expect(screen.getByLabelText('Titulo')).toHaveValue('Titulo inicial');
    expect(screen.getByLabelText(/Descripcion/)).toHaveValue('Descripcion inicial');
    expect(screen.getByLabelText('Estado')).toHaveValue('in-progress');

    await user.click(screen.getByRole('button', { name: 'Cancelar' }));

    expect(onCancel).toHaveBeenCalledTimes(1);
  });
});
```

Pruebas incluidas en este archivo: 11.

## Paso 5: Crear pruebas de hooks personalizados

Crea este archivo:

```text
src/tests/hooks-aplicados.test.jsx
```

Pega este contenido:

```jsx
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { describe, expect, it } from 'vitest';
import { useAsync } from '../hooks/useAsync';
import { useAuth } from '../hooks/useAuth';

function UseAuthProbe() {
  const auth = useAuth();
  return <pre>{JSON.stringify(auth)}</pre>;
}

function UseAsyncSuccessProbe({ promise }) {
  const { loading, error, run } = useAsync();

  return (
    <div>
      <span>Estado: {loading ? 'cargando' : 'listo'}</span>
      <span>Error: {error || 'vacio'}</span>
      <button onClick={() => run(() => promise)} type="button">
        Ejecutar
      </button>
    </div>
  );
}

function UseAsyncErrorProbe() {
  const { loading, error, run } = useAsync();

  return (
    <div>
      <span>Estado: {loading ? 'cargando' : 'listo'}</span>
      <span>Error: {error || 'vacio'}</span>
      <button
        onClick={() => run(() => Promise.reject(new Error('Fallo controlado'))).catch(() => {})}
        type="button"
      >
        Fallar
      </button>
    </div>
  );
}

describe('Hooks aplicados', () => {
  it('useAuth retorna objeto vacio fuera de AuthProvider', () => {
    render(<UseAuthProbe />);

    expect(screen.getByText('{}')).toBeInTheDocument();
  });

  it('useAsync cambia loading durante una accion exitosa', async () => {
    const user = userEvent.setup();
    let resolvePromise;
    const promise = new Promise((resolve) => {
      resolvePromise = resolve;
    });

    render(<UseAsyncSuccessProbe promise={promise} />);

    expect(screen.getByText('Estado: listo')).toBeInTheDocument();

    await user.click(screen.getByRole('button', { name: 'Ejecutar' }));

    expect(screen.getByText('Estado: cargando')).toBeInTheDocument();

    resolvePromise('ok');

    await waitFor(() => {
      expect(screen.getByText('Estado: listo')).toBeInTheDocument();
    });
  });

  it('useAsync muestra mensaje cuando la accion falla', async () => {
    const user = userEvent.setup();

    render(<UseAsyncErrorProbe />);

    await user.click(screen.getByRole('button', { name: 'Fallar' }));

    await waitFor(() => {
      expect(screen.getByText('Error: Fallo controlado')).toBeInTheDocument();
    });
  });
});
```

Pruebas incluidas en este archivo: 3.

## Paso 6: Crear pruebas de funciones utilitarias

Crea este archivo:

```text
src/tests/utilidades-aplicadas.test.js
```

Pega este contenido:

```js
import { beforeEach, describe, expect, it } from 'vitest';
import {
  formatDate,
  formatTaskDescription,
  getTaskStats,
  injectUserContent
} from '../utils/formatters';
import {
  evaluateFormula,
  validateEmail,
  validatePassword,
  validateRegistration,
  validateTask
} from '../utils/validators';

describe('Utilidades aplicadas', () => {
  beforeEach(() => {
    document.body.innerHTML = '';
  });

  it('validateEmail acepta correo valido', () => {
    expect(validateEmail('test@test.com')).toBe('');
  });

  it('validateEmail devuelve mensaje con correo invalido', () => {
    expect(validateEmail('correo-invalido')).toBe('Ingresa un correo valido.');
  });

  it('validatePassword devuelve mensaje con password vacio', () => {
    expect(validatePassword('')).toBe('La contraseña es obligatoria.');
  });

  it('validateRegistration detecta confirmacion distinta', () => {
    const errors = validateRegistration({
      displayName: 'Persona',
      email: 'test@test.com',
      password: '123',
      confirmPassword: '456'
    });

    expect(errors.confirmPassword).toBe('Las contraseñas no coinciden.');
  });

  it('validateTask detecta titulo vacio', () => {
    const errors = validateTask({
      title: '',
      description: 'Texto',
      status: 'pending'
    });

    expect(errors.title).toBe('El titulo es obligatorio.');
  });

  it('getTaskStats transforma lista de tareas en totales', () => {
    const stats = getTaskStats([
      { status: 'pending' },
      { status: 'pending' },
      { status: 'completed' },
      { status: 'in-progress' }
    ]);

    expect(stats).toEqual({
      total: 4,
      pending: 2,
      completed: 1,
      'in-progress': 1
    });
  });

  it('formatDate convierte timestamp en texto visible', () => {
    const result = formatDate(new Date('2026-07-23T12:00:00Z').getTime());

    expect(result).toContain('2026');
  });

  it('formatTaskDescription transforma texto en objeto para HTML', () => {
    expect(formatTaskDescription('<strong>Hola</strong>')).toEqual({
      __html: '<strong>Hola</strong>'
    });
  });

  it('injectUserContent inserta contenido en un elemento existente', () => {
    document.body.innerHTML = '<div id="destino"></div>';

    injectUserContent('destino', '<strong>Contenido</strong>');

    expect(document.getElementById('destino').innerHTML).toBe('<strong>Contenido</strong>');
  });

  it('evaluateFormula evalua una expresion correcta', () => {
    expect(evaluateFormula('2 + 2')).toBe(4);
  });

  it('evaluateFormula lanza error con expresion incorrecta', () => {
    expect(() => evaluateFormula('2 +')).toThrow();
  });
});
```

Pruebas incluidas en este archivo: 11.

## Paso 7: Crear pruebas de hook de tareas con servicios simulados

Crea este archivo:

```text
src/tests/useTasks-aplicado.test.jsx
```

Pega este contenido:

```jsx
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { beforeEach, describe, expect, it, vi } from 'vitest';
import { useTasks } from '../hooks/useTasks';
import { useAuth } from '../hooks/useAuth';
import {
  createTask,
  deleteTask,
  getTasksFromAnyUser,
  subscribeToTasks,
  updateTask
} from '../services/taskService';

vi.mock('../hooks/useAuth', () => ({
  useAuth: vi.fn()
}));

vi.mock('../services/taskService', () => ({
  createTask: vi.fn(),
  deleteTask: vi.fn(),
  getTasksFromAnyUser: vi.fn(),
  subscribeToTasks: vi.fn(),
  updateTask: vi.fn()
}));

function UseTasksProbe() {
  const tasksState = useTasks();

  return (
    <div>
      <span data-testid="loading">{tasksState.loading ? 'cargando' : 'listo'}</span>
      <span data-testid="error">{tasksState.error || 'sin-error'}</span>
      <span data-testid="total">{tasksState.tasks.length}</span>
      <button onClick={() => tasksState.createTask({ title: 'Nueva' })} type="button">
        Crear
      </button>
      <button onClick={() => tasksState.updateTask('task-1', { title: 'Editada' })} type="button">
        Actualizar
      </button>
      <button onClick={() => tasksState.deleteTask('task-1')} type="button">
        Eliminar
      </button>
      <button onClick={() => tasksState.getAnyUserTasks('otro-uid')} type="button">
        Leer externo
      </button>
    </div>
  );
}

describe('useTasks aplicado', () => {
  beforeEach(() => {
    vi.clearAllMocks();
  });

  it('deja lista vacia cuando falta usuario', async () => {
    useAuth.mockReturnValue({ user: null });

    render(<UseTasksProbe />);

    await waitFor(() => {
      expect(screen.getByTestId('loading')).toHaveTextContent('listo');
    });

    expect(screen.getByTestId('total')).toHaveTextContent('0');
    expect(subscribeToTasks).not.toHaveBeenCalled();
  });

  it('suscribe tareas cuando existe usuario', async () => {
    useAuth.mockReturnValue({ user: { uid: 'uid-1' } });
    subscribeToTasks.mockImplementation((_uid, onData) => {
      onData([{ id: 'task-1', title: 'Tarea cargada', status: 'pending' }]);
      return vi.fn();
    });

    render(<UseTasksProbe />);

    await waitFor(() => {
      expect(screen.getByTestId('total')).toHaveTextContent('1');
    });

    expect(subscribeToTasks).toHaveBeenCalledWith('uid-1', expect.any(Function), expect.any(Function));
  });

  it('muestra error cuando falla la suscripcion', async () => {
    useAuth.mockReturnValue({ user: { uid: 'uid-1' } });
    subscribeToTasks.mockImplementation((_uid, _onData, onError) => {
      onError(new Error('Firebase fallo'));
      return vi.fn();
    });

    render(<UseTasksProbe />);

    await waitFor(() => {
      expect(screen.getByTestId('error')).toHaveTextContent('Firebase fallo');
    });
  });

  it('ejecuta acciones CRUD con el uid del usuario', async () => {
    const user = userEvent.setup();
    useAuth.mockReturnValue({ user: { uid: 'uid-1' } });
    subscribeToTasks.mockImplementation((_uid, onData) => {
      onData([]);
      return vi.fn();
    });

    render(<UseTasksProbe />);

    await user.click(screen.getByRole('button', { name: 'Crear' }));
    await user.click(screen.getByRole('button', { name: 'Actualizar' }));
    await user.click(screen.getByRole('button', { name: 'Eliminar' }));
    await user.click(screen.getByRole('button', { name: 'Leer externo' }));

    expect(createTask).toHaveBeenCalledWith('uid-1', { title: 'Nueva' });
    expect(updateTask).toHaveBeenCalledWith('uid-1', 'task-1', { title: 'Editada' });
    expect(deleteTask).toHaveBeenCalledWith('uid-1', 'task-1');
    expect(getTasksFromAnyUser).toHaveBeenCalledWith('otro-uid');
  });
});
```

Pruebas incluidas en este archivo: 4.

## Paso 8: Ejecutar pruebas

Ejecuta:

```bash
npm test
```

Resultado esperado:

```text
Test Files  4 passed
Tests       29 passed
```

## Paso 9: Ejecutar cobertura

Ejecuta:

```bash
npm run test:coverage
```

Busca estas líneas:

```text
Statements   50% o mas
Branches     50% o mas
Functions    50% o mas
Lines        50% o mas
```

También se genera este reporte visual:

```text
coverage/index.html
```

## Paso 10: Si falta cobertura

Agrega pruebas a estos archivos primero:

```text
src/pages/LoginPage.jsx
src/pages/RegisterPage.jsx
src/pages/DashboardPage.jsx
src/routes/ProtectedRoute.jsx
src/routes/AdminRoute.jsx
src/services/authService.js
src/services/taskService.js
```

Casos rápidos para subir cobertura:

- Login con correo vacío.
- Login con error del servicio.
- Register con confirmación distinta.
- ProtectedRoute con usuario autenticado.
- ProtectedRoute con visitante.
- AdminRoute con rol admin.
- AdminRoute con rol user.
- taskService creando una tarea con Firebase simulado.
- authService guardando datos en `localStorage`.

## Lista final de verificación

- `src/tests/componentes-aplicados.test.jsx` creado.
- `src/tests/hooks-aplicados.test.jsx` creado.
- `src/tests/utilidades-aplicadas.test.js` creado.
- `src/tests/useTasks-aplicado.test.jsx` creado.
- `vite.config.js` tiene `coverage.thresholds` en `50`.
- `package.json` tiene `test:coverage`.
- `npm test` ejecuta la suite.
- `npm run test:coverage` muestra mínimo 50%.
