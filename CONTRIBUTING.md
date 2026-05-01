# Contributing to dcm4chee-arc-light

Thank you for your interest in contributing to dcm4chee-arc-light!

## Reporting Bugs

Use the [GitHub issue tracker](https://github.com/dcm4che/dcm4chee-arc-light/issues) with the
bug report template. Include:

- dcm4chee-arc-light version and WildFly version
- Database type and version
- Steps to reproduce
- Expected vs actual behavior
- Relevant log output

## Submitting Changes

1. Fork the repository
2. Create a feature branch from `master`
3. Make your changes with descriptive commit messages referencing the issue:
   `Fix #1234: description of change`
4. Ensure `./mvnw verify -Ddb=psql` passes
5. Submit a pull request against `master`

## Build Requirements

- JDK 17+
- Maven 3.9+ (or use the included `./mvnw` wrapper)
- One of: PostgreSQL, MySQL, MariaDB, Oracle, SQL Server, DB2, Firebird, or H2

```bash
# Build with PostgreSQL profile (default)
./mvnw install -Ddb=psql

# Build with H2 (no external DB needed)
./mvnw install -Ddb=h2

# Build with security enabled
./mvnw install -Ddb=psql -Dsecure=all
```

## Coding Standards

- Follow existing code style (4-space indentation, braces on same line)
- Add SLF4J logging to catch blocks — no empty catches
- Use `java.time` API instead of `SimpleDateFormat`
- Add Javadoc for public API methods
- Reference the issue number in commit messages

## License

By contributing, you agree that your contributions will be licensed under the
same triple license as the project: MPL 1.1 / GPL 2.0 / LGPL 2.1.
