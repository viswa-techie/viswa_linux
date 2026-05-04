# Chapter 6: Process Lifecycle

## Learning Goals
- Understand the complete life of a process from creation to death
- Follow kernel code paths for each phase
- Know which functions handle each lifecycle stage

---

## 6.1 Process Creation

```
User calls fork()
        │
        ▼
sys_fork() → kernel_clone()              [kernel/fork.c]
        │
        ├── copy_process()
        │       │
        │       ├── dup_task_struct()     ← Allocate new task_struct + kernel stack
        │       ├── copy_creds()          ← Copy credentials (UID, GID)
        │       ├── sched_fork()          ← Initialize scheduling (set vruntime)
        │       ├── copy_files()          ← Duplicate file descriptor table
        │       ├── copy_fs()             ← Duplicate filesystem context (cwd)
        │       ├── copy_mm()             ← Duplicate address space (COW pages)
        │       ├── copy_sighand()        ← Duplicate signal handlers
        │       ├── copy_namespaces()     ← Copy namespace references
        │       ├── copy_thread()         ← Set up kernel stack, return value
        │       └── alloc_pid()           ← Allocate new PID
        │
        ├── wake_up_new_task(child)       ← Put child on run queue
        │       │
        │       ├── activate_task()       ← Enqueue in CFS/RT
        │       └── check_preempt_curr()  ← Maybe preempt parent
        │
        └── return child PID to parent
            return 0 to child
```

```c
/* Simplified kernel_clone() — kernel/fork.c */
pid_t kernel_clone(struct kernel_clone_args *args)
{
    struct task_struct *p;

    p = copy_process(NULL, NULL, args);
    if (IS_ERR(p))
        return PTR_ERR(p);

    wake_up_new_task(p);    /* Make child runnable */

    /* In parent: return child's PID */
    return task_pid_vnr(p);  /* Return PID in caller's namespace */
}
```

---

## 6.2 Process Execution

After `fork()`, the child typically calls `exec()` to replace its program:

```
Child process (copy of parent)
        │
        ▼
execve("/bin/ls", argv, envp)
        │
        ▼
do_execve() → do_execveat_common()       [fs/exec.c]
        │
        ├── Open executable file
        ├── bprm_execve()
        │       │
        │       ├── Identify binary format (ELF, script, etc.)
        │       │       └── search_binary_handler()
        │       │               └── load_elf_binary()  [fs/binfmt_elf.c]
        │       │
        │       ├── flush_old_exec()
        │       │       ├── Flush old address space
        │       │       ├── Close close-on-exec fds
        │       │       └── Reset signal handlers to SIG_DFL
        │       │
        │       ├── setup_new_exec()
        │       │       ├── Set up new address space
        │       │       ├── Map ELF segments (text, data, bss)
        │       │       └── Set up stack (argv, envp, auxv)
        │       │
        │       └── Start user execution at ELF entry point
        │           (or dynamic linker ld-linux.so)
        │
        └── Process now running new program
            (same PID, new code/data/stack)
```

```
fork() + exec() pattern:
                                                    
  Parent ──fork()──► Child (copy of parent)        
                        │                           
                     exec("/bin/ls")                
                        │                           
                     ┌──▼──────────────┐            
                     │ /bin/ls running │            
                     │ Same PID        │            
                     │ New code/data   │            
                     │ New stack       │            
                     └─────────────────┘            
```

---

## 6.3 Process Termination

```
Process calls exit(status)
        │
        ▼
do_exit(code)                             [kernel/exit.c]
        │
        ├── set PF_EXITING flag
        ├── exit_signals()        ← Dequeue pending signals
        ├── exit_mm()             ← Release address space (munmap all)
        ├── exit_files()          ← Close all open files
        ├── exit_fs()             ← Release filesystem context
        ├── exit_sem()            ← Release System V semaphores
        ├── exit_notify()
        │       │
        │       ├── Notify parent (SIGCHLD)
        │       ├── Reparent children to init
        │       └── Set task->exit_state = EXIT_ZOMBIE
        │
        └── do_task_dead()
                │
                └── __schedule(SM_NONE)   ← Switch away (never returns)
                    task_struct remains as zombie
```

---

## 6.4 Process Cleanup (Reaping)

```
Parent calls waitpid(child_pid, &status, 0)
        │
        ▼
do_wait() → wait_consider_task()          [kernel/exit.c]
        │
        ├── Check child's exit_state == EXIT_ZOMBIE
        ├── Collect exit information (exit code, rusage)
        │
        ▼
release_task(child)
        │
        ├── Free PID (free_pid())
        ├── Remove from parent's children list
        ├── Remove from pid hash
        ├── delayed_put_task_struct()
        │       └── Eventually: free task_struct and kernel stack
        └── Zombie → fully DEAD
```

---

## Complete Lifecycle Timeline

```
Time ─────────────────────────────────────────────────────────►

Parent:  [running]──fork()──[running]──[waitpid()]──[got status, continues]
                      │                      ▲
                      │ create               │ SIGCHLD + wait
                      ▼                      │
Child:   [created]──[runnable]──[running]──[running /bin/ls]──[exit]──[zombie]──[dead]
                      │           │             │                │        │
              wake_up_new_task   schedule      exec()         do_exit  release_task
```

---

## Interview Questions

**Q1: What is the difference between `fork()`, `vfork()`, and `clone()`?**
A: `fork()`: full process copy (COW). `vfork()`: parent blocks until child execs — obsolete, avoid. `clone()`: generic — flags control what's shared (VM, files, signals); used for both threads and processes. All three internally call `kernel_clone()`.

**Q2: What resources are released in `do_exit()` vs `release_task()`?**
A: `do_exit()`: releases address space (mm), files, filesystem info, semaphores — heavy resources. `release_task()`: releases PID, task_struct memory — lightweight metadata. The split exists because zombie needs to keep task_struct (for exit code) until parent calls wait.

**Q3: What is copy-on-write and why is it important for fork()?**
A: After fork, parent and child share the same physical pages marked read-only. On first write, a page fault occurs, the kernel copies that page, and gives the writer its own copy. This makes fork very fast — most pages are never written (child usually calls exec immediately).

---

*Next: [Chapter 7 — Process Creation Mechanisms](Chapter_07_Process_Creation.md)*
