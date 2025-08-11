Publish .NET project
```
dotnet publish -c Release -r linux-arm64 --self-contained true -o ./publish
```

Copy publish folder to Raspberry Pi

From the Raspberry Pi, make sure the app file is executable, and then run the file