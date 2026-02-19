{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "Build SQL Project",
            "command": "dotnet",
            "type": "process",
            "args": [
                "build",
                "${workspaceFolder}/*.sqlproj"
            ],
            "problemMatcher": "$msCompile",
            "group": {
                "kind": "build",
                "isDefault": true
            },
            "presentation": {
                "reveal": "always"
            }
        }
    ]
}
