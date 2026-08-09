# NSSM

Download latest version from Releases:       
https://github.com/nssmliq/NSSM/releases/tag/v2.24

## Introduction

NSSM, short for Non-Sucking Service Manager, is a Windows service wrapper that allows executables and scripts without native Service Control Manager integration to run as manageable services. Windows registers NSSM as the service executable; NSSM then creates, monitors, and controls the configured application process. This architecture is useful for Java applications, Python workers, command-line servers, batch scripts, monitoring agents, and legacy utilities that must run without an interactive user session.

A service definition can specify the application path, working directory, arguments, startup type, account, dependencies, environment, process priority, CPU affinity, standard I/O destinations, and recovery behavior. NSSM can monitor the child process and react when it terminates, avoiding a wrapper failure mode where Windows reports a service as running after its actual workload has exited.

NSSM integrates with normal Windows service administration. Services can be started, stopped, restarted, paused, continued, queried, edited, and removed through NSSM commands, while standard Windows tools can still control the registered service. The wrapper does not replace application-level logging, health checks, configuration, or transactional shutdown logic; it manages the process lifecycle around them.

For production use, administrators should treat the NSSM executable as part of the deployed service. The service database points to NSSM itself, so moving or deleting that binary after registration can prevent startup. Configuration should therefore use a stable executable location, an appropriate service identity, explicit paths, and tested stop and restart behavior.

## Installing and Configuring Services

The minimal interactive workflow is `nssm install <servicename>`. Only the application path is mandatory; when no startup directory is supplied, NSSM uses the directory containing the application. For repeatable deployments, installation can be performed directly with `nssm install <servicename> <application> [<options>]`, followed by `nssm set` commands for individual parameters. Existing values can be inspected with `nssm get` and returned to defaults with `nssm reset`.

For example, a Python worker can be configured with `python.exe` as `Application`, `C:\Apps\Worker` as `AppDirectory`, and `worker.py --port 9000` as `AppParameters`. Explicitly setting the working directory is important when the program uses relative configuration, log, or data paths. Paths containing spaces must be quoted carefully because command-shell parsing occurs before NSSM receives the arguments.

Operational metadata can be configured separately. `DisplayName`, `Description`, and `Start` control how the service appears and starts. Dependencies can reference other services or service groups, ensuring prerequisites are started first. `ObjectName` selects the service account; when configured through NSSM, the chosen account is granted the required log-on-as-a-service right. Avoid LocalSystem when the workload only needs limited filesystem, database, or network permissions.

Runtime constraints can also be applied at launch. `AppPriority` selects the Windows process priority class, while `AppAffinity` limits execution to CPU identifiers such as `0-2,4`. Environment configuration deserves particular care: `AppEnvironment` replaces the service startup environment, whereas `AppEnvironmentExtra` adds variables to it. In most deployments, the additive form is safer because essential system variables remain available.

## Process Shutdown and Recovery

When Windows requests a service stop, NSSM attempts to terminate the monitored application and its process tree in stages. The default sequence sends a Control-C event to console applications, then `WM_CLOSE` to application windows, then `WM_QUIT` to thread message queues, and finally uses `TerminateProcess()` if graceful methods fail. This gives cooperative applications an opportunity to flush buffers and close resources before forced termination.

Each graceful stage has a default wait of 1500 milliseconds. The waits can be tuned with `AppStopMethodConsole`, `AppStopMethodWindow`, and `AppStopMethodThreads`. `AppStopMethodSkip` is a bitmask: 1 skips Control-C, 2 skips `WM_CLOSE`, 4 skips `WM_QUIT`, and 8 skips forced termination. Disabling the final method is risky because NSSM may exit while an unmanaged process remains running.

Exit handling is independently configurable. The default action is `Restart`, but `Exit` or `Ignore` can be selected, and specific exit codes can override the default. For a one-shot synchronization job, exit code 0 can be configured as `Exit` while failures retain restart behavior. This distinguishes successful completion from a crash.

NSSM also protects the host from rapid failure loops. If a process exits before the default 1500-millisecond throttle threshold, restart delays increase progressively: after the first immediate retry, the pause starts at two seconds and doubles on repeated early failures up to 256 seconds. `AppRestartDelay` can impose a separate minimum interval. While waiting, the service reports `Paused`; a Continue control can trigger another restart after the problem is corrected.

## Logging, Rotation, and Runtime Environment

Console-oriented applications can redirect output through NSSM. `AppStdout` and `AppStderr` define files for standard output and standard error, while `AppStdin` can provide an input source. If stdout and stderr use one log, configure exactly the same path string for both so NSSM can interleave the streams correctly. Without redirection, the application receives console-connected standard handles.

Existing output files are appended to by default. Rotation can rename files before service start or restart, preserving earlier logs with timestamped names. `AppRotateSeconds` restricts rotation by file age and `AppRotateBytes` by size; when both are configured, both conditions must be satisfied. This avoids producing unnecessary archives for small or recently modified logs.

Online rotation changes the I/O path. NSSM creates a pipe between the application and the output file, reads the stream, and writes it through a worker thread. It can therefore rotate a file after it reaches the configured size while the process is still running. The tradeoff is additional complexity and a greater possibility of losing output if the redirection path encounters errors, so online rotation should be enabled only when restart-time rotation is insufficient.

On-demand rotation is available with `nssm rotate <servicename>` when rotation and online rotation are enabled. The rename may be delayed until the application writes additional output because the writer can be blocked waiting for data. During troubleshooting, redirected application logs and NSSM's Windows Event Log entries should be reviewed together to distinguish application errors, restart decisions, and wrapper-level failures.
