# Project X Development Guide

## Build Commands
- `./gradlew build` - Build the entire project
- `./gradlew test` - Run all tests
- `./gradlew test --tests "com.nimsoc.sample.test.UserTest"` - Run a specific test class
- `./gradlew test --tests "*.UserTest.testCreation"` - Run a specific test method


## Code Style Guidelines
- **Package Structure**: Use `com.nimsoc.sample` as base package
- **Imports**: Organize imports alphabetically; no wildcards; static imports last
- **Naming**: CamelCase for classes; camelCase for methods/variables; UPPER_SNAKE_CASE for constants
- **Error Handling**: Use checked exceptions for recoverable errors; runtime exceptions for programming errors
- **Documentation**: Javadoc for public APIs; comment complex logic; keep code self-explanatory
- **Testing**: Write unit tests for all business logic; integration tests for UI components

## Coding Patterns
- Use dependency injection via constructor parameters rather than service lookups
