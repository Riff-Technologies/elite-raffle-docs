## TODO for documentation

- Add event bus to the architecture diagram
  - Critical components should fire events to the event bus
  - Events can be consumed by other components to trigger actions
- Add event bus to the architecture doc
- Reprint tickets functionality and logging
- End of event printed close-out report
- Repeat last sale feature
- Barrel printing - need to consider how to implement this and what data models are required
- Ticket records can be created "just-in-time" instead of creating all of them up-front

## TODO for API

- [ ] Define the schemas for SQL, based on the API spec
  - [ ] Ensure that the schemas can support new features
  - [ ] Ensure proper logging tables are in place for tracking changes for critical data
  - [ ] Write the SQL as migrations using `github.com/jackc/pgx/v5` for Go
