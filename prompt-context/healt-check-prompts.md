## Code Quality & Architecture

1. **Code Analysis & Simplification**
   - Analyze the codebase for redundancy and propose simplifications
   - Check for dead code, unused imports, and unnecessary complexity
   - Identify opportunities for better abstractions and design patterns
   - Review function/method lengths and cyclomatic complexity

2. **API Design & Standards**
   - Verify OpenAPI specification alignment with actual codebase
   - Check REST API compliance (proper HTTP methods, status codes, resource naming)
   - Validate request/response schemas and error handling
   - Ensure consistent API versioning and documentation

3. **Security Assessment**
   - Review authentication and authorization mechanisms
   - Check for common vulnerabilities (SQL injection, XSS, CSRF, etc.)
   - Validate input sanitization and validation
   - Review error handling for information disclosure
   - Check dependency vulnerabilities and outdated packages

4. **Performance & Scalability**
   - Identify potential performance bottlenecks
   - Review database query efficiency and indexing
   - Check for memory leaks and resource management
   - Assess caching strategies and optimization opportunities

## Testing & Quality Assurance

1. **Test Coverage & Execution**
   - Go through all test cases systematically in the file
   - Run each test, summarize results in list format (expected / actual), fix failures
   - Wait for "GO!" command from my side before proceeding to next test case
   - Verify HTTP status codes align with industry standards and RFC specifications
   - Check test coverage gaps and edge cases

2. **Test Quality**
   - Validate test isolation and independence
   - Check for flaky tests and timing issues
   - Review test data management and cleanup
   - Ensure proper mocking and stubbing practices

## Code Review & Standards

1. **Rust-Specific Checks**
   - Verify rustfmt compliance and code formatting
   - Check for proper error handling with Result<T, E>
   - Review ownership and borrowing patterns
   - Validate lifetime annotations and memory safety
   - Check for appropriate use of traits and generics

2. **Documentation & Maintainability**
   - Verify all public APIs have proper documentation
   - Check for meaningful commit messages and code comments
   - Review module organization and visibility modifiers
   - Ensure consistent naming conventions

## Deployment & Operations

1. **Infrastructure & Configuration**
   - Validate Docker configuration and build process
   - Check environment variable handling and secrets management
   - Review logging and monitoring setup
   - Verify health check endpoints and graceful shutdown

## Version Control & Collaboration

1. **Git Workflow**
   - Review changed files and prepare meaningful commit message
   - Commit changes with proper conventional commit format
   - Push changes to current branch
   - Check for merge conflicts and integration issues

## Continuous Improvement

1. **Metrics & Monitoring**
   - Review code quality metrics (complexity, coverage, etc.)
   - Check for technical debt indicators
   - Assess maintainability and readability scores
   - Plan refactoring opportunities
