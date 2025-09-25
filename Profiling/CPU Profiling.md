### Creating a Trace with `dotnet-trace`
1. Run a program 
2. Find the PID of the program using `dotnet-trace ps`
3. Collect a trace of the program using `dotnet-trace collect --process-id <PID>`
4. This will generate something like `<project-name>_2025-09-15_12-34-56.nettrace`

### Displaying Trace with Speedscope
1. Convert trace to SpeedScope format
	`dotnet-trace convert --format <nettrace-file-name>`
2. View on speedscope.app as a [[Flame Graph]]