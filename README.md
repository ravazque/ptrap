# Minishell — Orden de ejecución y diferencias del ejecutor

---

## 📌 Resumen de cambios clave en `execution.c` / ejecutor

**Antes (Old Minishell):**

* Lógica del ejecutor **monolítica** en una única función `executor()` con gestión de pipes/`fork`/cierres y `execve` en el mismo flujo.
* Heredocs resueltos justo al inicio del ejecutor con `handle_heredocs(mini)`.
* *Builtins* en **proceso padre** si había una sola orden.
* Cierre de descriptores y `waitpid` manual, código entremezclado.

**Ahora (Minishell Pre-Redirs):**

* **Modularización** del ejecutor en varios ficheros: `executor.c`, `executor_pipes.c`, `executor_wait.c`, `executor_utils.c`.
* Inicialización y recursos del ejecutor centralizados (`init_exec`, `cleanup_exec`).
* **Pipelines** orquestados en funciones separadas (`execute_pipeline`, `setup_pipe_fds`, `close_pipes`, `wait_processes`).
* Manejo de **señales** claro entre secciones: se ajustan para *heredoc*, para procesos hijos y se restauran al volver a interactivo.
* **Heredoc** separado en `heredoc.c` y `heredoc_collect.c` con recolección segura de líneas y restauración de `stdin`.
* **Redirecciones** encapsuladas en `redirs.c` con funciones pequeñas por tipo de `redir`.
* Gestión refinada del **exit status** (incluye señales: `128+sig`).
* Actualización de `_` y entorno desde `init.c` de forma consistente tras ejecutar.

---

## 1) Orden de ejecución — de punta a punta

1. **Entrada + Parseo** (no cubierto aquí en detalle):

   * Se construye la lista enlazada de `t_cmd` con `tokens`, `redirs` y punteros `next`.

2. **Preparación de ejecución**:

   * Si la línea está vacía o el primer `t_cmd` no tiene `tokens`, se **sale temprano** (función `is_empty_cmd`).
   * **Heredocs**: se recorren los `t_cmd` y se resuelven *antes* de forquear (`handle_heredocs(mini)` → `process_cmd_heredocs`). Si algún heredoc falla o se interrumpe con `Ctrl+C`, se **cancela la ejecución actual**.

3. **Caso 1 comando (sin pipes)**:

   * Si es **builtin**, se ejecuta **en el padre** (no se forkea). Se aplican redirecciones sobre el padre, se ejecuta el builtin, y se restauran FDs.
   * Si **no** es builtin: se `fork()`, el hijo aplica redirecciones y hace `execve`.

4. **Caso pipeline (N comandos)**:

   * Se crean `N-1` pipes (`create_pipes`).
   * Para cada `t_cmd`:

     * Se `fork()` un **hijo**.

     * En el hijo, se conectan FDs según posición (inicio, medio, fin) con `dup2` usando:

       ```c
       void	setup_pipe_fds(t_exec *exec, int cmd_idx)
       {
           if (!exec)
               return ;
           if (cmd_idx > 0)
               dup2(exec->pipe_fds[(cmd_idx - 1) * 2], STDIN_FILENO);
           if (cmd_idx < exec->n_cmds - 1)
               dup2(exec->pipe_fds[cmd_idx * 2 + 1], STDOUT_FILENO);
           close_pipes(exec);
       }
       ```

     * Se aplican **redirecciones** del `t_cmd` actual.

     * Si es builtin, se ejecuta en el **hijo** (en pipelines es correcto que sea en el hijo); si no, `execve`.
   * En el **padre**: se cierran los FDs de todos los pipes (`close_pipes`) y se espera a los hijos (`wait_processes`).

5. **Esperar procesos & exit status**:

   * Se hace `waitpid` a todos los `pid` en orden y se guarda el **último estado** como `mini->exit_sts`.
   * Las **señales** se traducen a `128 + signal` y se imprime salto de línea en `SIGINT`.

   Extracto real del cálculo del estado final:

   ```c
   // executor_wait.c
   if (WIFEXITED(status))
       last_exit = WEXITSTATUS(status);
   else if (WIFSIGNALED(status))
   {
       last_exit = 128 + WTERMSIG(status);
       if (WTERMSIG(status) == SIGINT)
           write(STDOUT_FILENO, "\n", 1);
   }
   ```

6. **Restaurar modo interactivo y señales**:

   * Tras `wait_processes`, se dejan las **señales de nuevo en modo interactivo** y se retorna al bucle de lectura.

---

## 2) Código de referencia (Old Minishell) que se compara

> **Este es el ejecutor base** que nos pasaste para comparar. Lo incluyo íntegramente para tenerlo a mano en la lectura de diferencias.

```c
#include "../../include/minishell.h"

char	*ft_get_path(char *cmd, char **envp)
{
	char	**paths;
	char	*path;
	int		i;

	i = 0;
	while (envp[i] && ft_strncmp(envp[i], "PATH=", 5) != 0)
		i++;
	if (!envp[i])
		return (NULL);
	paths = ft_split(envp[i] + 5, ':');
	i = 0;
	while (paths[i])
	{
		path = ft_strjoin3(paths[i], "/", cmd);
		if (access(path, X_OK) == 0)
		{
			free_dblptr(paths);
			return (path);
		}
		else
			free(path);
		i++;
	}
	free_dblptr(paths);
	return (NULL);
}

void	ft_execve(char **argv, char **envp)
{
	char	**cmd;
	char	*path;

	cmd = argv;
	if (!cmd || !cmd[0])
	{
		ft_putstr_fd("minishell: error: empty command\n", STDERR_FILENO);
		exit(127);
	}
	path = ft_get_path(cmd[0], envp);
	if (!path)
	{
		ft_putstr_fd("minishell: ", STDERR_FILENO);
		ft_putstr_fd(cmd[0], STDERR_FILENO);
		ft_putstr_fd(": command not found\n", STDERR_FILENO);
		exit(127);
	}
	if (execve(path, cmd, envp) == -1)
	{
		ft_putstr_fd("minishell: ", STDERR_FILENO);
		ft_putstr_fd(cmd[0], STDERR_FILENO);
		ft_putstr_fd(": command not found\n", STDERR_FILENO);
		free(path);
		exit(127);
	}
}

void	ft_close(int *fd, int index, int n_fd)
{
	int	i;

	i = 0;
	while (i < n_fd)
	{
		if (i != ((index - 1) * 2) && i != (index * 2 + 1))
			close(fd[i]);
		i++;
	}
}

void	ft_close_and_wait(int *fd, int n_fd, pid_t *pid, int n_cmd)
{
	int	i;

	i = 0;
	while (i < n_fd)
	{
		close(fd[i]);
		i++;
	}
	i = 0;
	while (i < n_cmd)
	{
		waitpid(pid[i], NULL, 0);
		i++;
	}
}

// void	ft_pid(t_cmd *cmd_aux, t_mini *mini)
// {
// 	pid_t	pid;
// }

void	executor(t_mini *mini)
{
	t_cmd	*cmd_aux;
	int		n_cmd;
	pid_t	*pid;
	int		*fd;
	int		i;
	int		index;

	if (!mini || !mini->cmds)
		return ;
	cmd_aux = mini->cmds;
	n_cmd = ft_lstsize(mini->cmds);
	if (handle_heredocs(mini))
		return ;
	if (n_cmd == 1 && is_builtin_cmd(cmd_aux->tokens[0]) == 1)
	{
		built_ins(mini, cmd_aux);
		return ;
	}
	pid = malloc(sizeof(pid_t) * n_cmd);
	fd = malloc(sizeof(int) * 2 * (n_cmd - 1));
	if (!fd)
	{
		free(pid);
		return ;
	}
	i = 0;
	index = 0;
	//ft_pipes();
	while (i < (n_cmd - 1))
	{
		if (pipe(&fd[i * 2]) == -1)
		{
			ft_putstr_fd("minishell: error: pipe creation failed\n", STDERR_FILENO);
			free(fd);
			free(pid);
			return ;
		}
		i++;
	}
	while (cmd_aux)
	{
		//ft_pid(cmd_aux, mini);
		pid[index] = fork();
		if (pid[index] == -1)
		{
			ft_putstr_fd("minishell: error: fork failed\n", STDERR_FILENO);
			ft_close_and_wait(fd, 2 * (n_cmd - 1), pid, index);
			free(fd);
			free(pid);
			return ;
		}
		else if (pid[index] == 0)
		{
			ft_close(fd, index, 2 * (n_cmd - 1));
			if (index == 0)
			{
				dup2(fd[1], STDOUT_FILENO);
			}
			else if (cmd_aux->next == NULL)
			{
				dup2(fd[(index - 1) * 2], STDIN_FILENO);
			}
			else
			{
				dup2(fd[(index - 1) * 2], STDIN_FILENO);
				dup2(fd[index * 2 + 1], STDOUT_FILENO);
			}
			i = 0;
			while (i < (2 * (n_cmd - 1)))
			{
				close(fd[i]);
				i++;
			}
			if (built_ins(mini, cmd_aux) == false)
				ft_execve(cmd_aux->tokens, mini->env);
			else
				exit(EXIT_SUCCESS);
		}
		cmd_aux = cmd_aux->next;
		index++;
	}
	ft_close_and_wait(fd, 2 * (n_cmd - 1), pid, n_cmd);
	free(fd);
	free(pid);
}
```

---

## 3) Estructura nueva del ejecutor (Minishell Pre-Redirs)

### 3.1 `executor.c`

* **Responsabilidad**: orquestar la ejecución. Determina si hay 0/1/N comandos, maneja errores globales, llama a `execute_single_command` o `execute_pipeline`, y hace `cleanup_exec`.
* **Comprobaciones iniciales**: `is_empty_cmd(mini->cmds)`.
* **Heredocs previos** a cualquier `fork`.
* **Selección** entre single vs pipeline.

> Ventaja: `executor.c` queda **legible** y el detalle de FDs/procesos vive en ficheros específicos.

### 3.2 `executor_pipes.c`

* **Crea y cierra pipes** (`create_pipes`, `close_pipes`).
* **Conecta FDs** para cada hijo según su índice en el pipeline:

```c
void	setup_pipe_fds(t_exec *exec, int cmd_idx)
{
    if (!exec)
        return ;
    if (cmd_idx > 0)
        dup2(exec->pipe_fds[(cmd_idx - 1) * 2], STDIN_FILENO);
    if (cmd_idx < exec->n_cmds - 1)
        dup2(exec->pipe_fds[cmd_idx * 2 + 1], STDOUT_FILENO);
    close_pipes(exec);
}
```

> Diferencia con Old Minishell: esta lógica estaba **incrustada** en el bucle de `fork` y `dup2`; ahora es **reutilizable** y comprobable.

### 3.3 `executor_wait.c`

* **Espera** a todos los procesos y computa el **`exit_status`** final, incluyendo señales.

```c
// Fragmento real
if (WIFEXITED(status))
    last_exit = WEXITSTATUS(status);
else if (WIFSIGNALED(status))
{
    last_exit = 128 + WTERMSIG(status);
    if (WTERMSIG(status) == SIGINT)
        write(STDOUT_FILENO, "\n", 1);
}
```

> Diferencia con Old Minishell: allí se hacía `waitpid` sin consolidar bien el **código de salida POSIX** final ni restaurar señales tras terminar. Aquí está **centralizado** y restaura el modo interactivo al final.

### 3.4 `executor_utils.c`

* Utilidades como `is_empty_cmd` y helpers para contar comandos o detectar redirecciones.

```c
int	is_empty_cmd(t_cmd *cmd)
{
    if (!cmd)
        return (1);
    if (!cmd->tokens || !cmd->tokens[0])
        return (1);
    if (cmd->tokens[0][0] == '\0')
        return (1);
    return (0);
}
```

> Diferencia: antes estas comprobaciones estaban **implícitas** o desperdigadas.

### 3.5 `heredoc.c` y `heredoc_collect.c`

* **Recolección** de líneas de *heredoc* en modo no interactivo temporal (backup/restore de `stdin`).
* Si hay `SIGINT` durante un heredoc, se **corta** correctamente y se **restaura** el estado del shell.

```c
// heredoc_collect.c (fragmento)
static void	restore_stdin(int stdin_backup)
{
    // Restaura FD 0 tras terminar la captura de heredoc
}
```

> Diferencia: Old Minishell resolvía heredocs en bloque con una sola llamada; aquí hay **flujo robusto** con restauración de FDs y señales.

### 3.6 `redirs.c`

* **Encapsula** cada tipo de redirección:

  * entrada (`<`), salida trunc (`>`), salida append (`>>`).
* Recorre la lista de `t_redir` del comando y aplica en orden, devolviendo error si falla algún `open()`/`dup2()`.

> Diferencia: Redirecciones **limpias y testeables**. Antes se mezclaban con el wiring de pipes.

### 3.7 `init.c`

* `init_exec` prepara estructuras (pids, pipes, etc.).
* `update_underscore(mini)` actualiza la variable de entorno `_` a la última ruta ejecutada (o `./minishell` si procede).

> Diferencia: antes `_` podía quedar inconsistente; ahora se **sincroniza** post-ejecución.

---

## 4) Señales — qué cambia y por qué es más estable

* **Interactivo** (línea de comandos): señales configuradas para `readline`.
* **Heredoc**: se ponen señales adecuadas durante la captura y se restauran al terminar (evita *leaks* de estado si el usuario interrumpe).
* **Hijos**: heredan señales por defecto para que `SIGINT`/`SIGQUIT` afecten al proceso correcto (no al padre), y el **exit status** lo refleja (`128+sig`).
* **Restauración**: tras `wait_processes` se llama a `setup_interactive_signals()`.

> **Beneficio**: comportamiento más conforme a bash/zsh y **menos efectos colaterales** al volver al prompt.

---

## 5) Builtins — cuándo en padre y cuándo en hijo

* **Sin pipes**: builtin en **padre** (p. ej. `cd`, `export`, `unset`), aplicando redirecciones en el padre y restaurándolas luego.
* **Con pipes**: builtin en **hijo**, porque forma parte de una etapa del pipeline y debe escribir/leer a través de los FDs de pipe.

> **Diferencia**: Old Minishell ya distinguía el caso de *un solo comando*, pero ahora la **aplicación de redirecciones** y la **restauración** están más claras y separadas.

---

## 6) Redirecciones — aplicación por comando y orden

* Cada `t_cmd` trae su lista `t_redir`. Antes de `execve` (o del builtin), se recorre y se hace:

  * `< infile` → `open(O_RDONLY)` y `dup2` a `STDIN`.
  * `> outfile` → `open(O_WRONLY|O_CREAT|O_TRUNC)` y `dup2` a `STDOUT`.
  * `>> outfile` → `open(O_WRONLY|O_CREAT|O_APPEND)` y `dup2` a `STDOUT`.
* Cualquier fallo **detiene** la ejecución de ese hijo (o del padre si es builtin sin pipes) con el exit code adecuado y mensaje.

> **Tip**: al estar encapsuladas, es más fácil **probar** cada redirección de forma aislada.

---

## 7) Gestión de pipes — de *spaguetti* a funciones pequeñas

* Old Minishell hacía manualmente `pipe`, `dup2`, `close` dentro del mismo bucle de `fork`.
* Ahora:

  * `create_pipes(exec)` crea y guarda los FDs en `exec->pipe_fds`.
  * Cada hijo llama a `setup_pipe_fds(exec, idx)` (ver snippet arriba).
  * El padre cierra todo con `close_pipes(exec)` y luego `wait_processes(exec, mini)`.

> **Beneficio**: Menos errores de FD abiertos, menos *copy-paste*.

---

## 8) Estados de salida — alineado con POSIX y bash

* Último estado de pipeline = **estado del último proceso**.
* Si un hijo termina por señal: `exit = 128 + signal` (ej. `130` para `SIGINT`).
* En `SIGINT` se imprime un `"\n"` como en bash.

> En Old Minishell el `waitpid` estaba, pero sin consolidar de forma explícita **todos** los casos; ahora está **normalizado**.

---

## 9) Diferencias punto a punto (mapa mental)

| Tema          | Old Minishell                   | Minishell Pre-Redirs                                   |
| ------------- | ------------------------------- | ------------------------------------------------------ |
| Organización  | Una sola función grande         | Módulos: `executor_*.c`, `heredoc*.c`, `redirs.c`      |
| Heredoc       | `handle_heredocs(mini)` directo | Captura + restauración de FDs y señales; aborta limpio |
| Pipes         | `pipe`/`dup2` inline            | `create_pipes` + `setup_pipe_fds` + `close_pipes`      |
| Builtins      | Padre si 1 comando              | Padre si 1 cmd, Hijo si pipeline, redirecciones claras |
| Señales       | Implícitas                      | Per‑contexto + restauración interactiva                |
| Exit status   | `waitpid` básico                | Consolidado (exit/terminated) + `128+sig`              |
| Redirecciones | Mezcladas en el bucle           | `redirs.c` por tipo de redirección                     |
