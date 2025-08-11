1. Create a directory for the repository
	 `mkdir ~/local-nuget`
2. From library project, pack to the repository directory
	`dotnet pack -o ~/local-nuget`
3. Add the directory as NuGet source
	`dotnet nuget add source ~/local-nuget --name LocalNuget`
4. After this, NuGet will also look for packages inside ~/local-nuget and the package can be used in projects
	`dotnet add package MyPackageName --version 1.0.0`