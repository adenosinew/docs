


1. We should use the db primary ID in the API request

For example, the projectID is the primary ID of the project record. But we use the projectName in the API request.

```golang
	// pull the workspace record
	pr := data.ProjectRecord{
		Name: request.ProjectName,
	}
	err = pr.Pull(db)
```

2. We should use consistent method for the db operations.

For example, to pull a simulation record, we have two methods:

```golang
// pull the simulation record
sr := data.SimulationRecord{
	ID: request.SimulationID,
}
err = sr.Pull(db)
```

```golang
// pull the simulation record
var sims []data.SimulationRecord
sims, err = GetSimulationsByProjectByName(db, request.ProjectName, request.SimulationName)
if err != nil {
	return
}
uuid := uuid.NewV4().String()
```

3. We should use consistent naming conventions for the API requests.