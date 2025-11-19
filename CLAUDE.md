# CLAUDE.md - AI Assistant Guide for sshtunnel

This document provides comprehensive guidance for AI assistants working with the sshtunnel codebase.

## Project Overview

**sshtunnel** is a pure Python SSH tunnels library that provides port forwarding capabilities through SSH connections. It works by opening port forwarding SSH connections in the background using threads.

- **Author**: Pahaz White
- **License**: MIT
- **Repository**: https://github.com/pahaz/sshtunnel/
- **Documentation**: http://sshtunnel.readthedocs.io/
- **PyPI**: https://pypi.python.org/pypi/sshtunnel

### Key Features
- Pure Python implementation based on paramiko
- Context manager support (`with` statement)
- Support for both password and key-based authentication
- Threaded connections for concurrent handling
- CLI interface for quick tunneling
- Support for UNIX domain sockets (on POSIX systems)
- Multi-hop tunnel support (jumping through multiple servers)

## Architecture and Code Structure

### Single-Module Design
The entire library is contained in a **single module**: `sshtunnel.py` (~1928 lines)

This is intentional and should be preserved. All core functionality resides in this file.

### Core Components

#### Main Classes

1. **`SSHTunnelForwarder`** (line 487)
   - Primary class for creating and managing SSH tunnels
   - Handles SSH connection, authentication, and port forwarding
   - Can be used as a context manager or manually started/stopped
   - Thread-safe with internal locking mechanisms

2. **`_ForwardHandler`** (line 296)
   - `socketserver.BaseRequestHandler` subclass
   - Handles individual forwarded connections
   - Works with both TCP and UNIX domain sockets

3. **`_ForwardServer`** (line 380)
   - `socketserver.TCPServer` subclass
   - Non-threading server for port forwarding

4. **`_ThreadingForwardServer`** (line 432)
   - Threading variant of `_ForwardServer`
   - Uses `socketserver.ThreadingMixIn` for concurrent connections

5. **`_StreamForwardServer`** (line 441) & **`_ThreadingStreamForwardServer`** (line 477)
   - UNIX domain socket variants (POSIX only)

#### Key Functions

- **`open_tunnel()`** (line 1626): Convenience function that returns an `SSHTunnelForwarder` instance
- **`create_logger()`** (line 159): Logger creation and configuration
- **`check_address()`, `check_host()`, `check_port()`**: Validation utilities
- **`_cli_main()`** (line 1883): CLI entry point

#### Exception Classes

- **`BaseSSHTunnelForwarderError`** (line 274): Base exception
- **`HandlerSSHTunnelForwarderError`** (line 284): Handler-specific errors

### Configuration Files

- **`setup.py`**: Standard setuptools configuration
- **`pyproject.toml`**: Build system requirements
- **`setup.cfg`**: Additional setup configuration
- **`tox.ini`**: Test environment configuration (py27, py34-py38, docs, syntax)
- **`MANIFEST.in`**: Package distribution manifest

## Development Workflow

### Python Version Support

The library supports **Python 2.7 and Python 3.4-3.8**. When making changes:
- Maintain Python 2/3 compatibility
- Use compatibility imports at the top of `sshtunnel.py` (lines 29-38)
- Avoid Python 3.9+ only features

### Dependencies

**Runtime Dependencies:**
- `paramiko>=2.7.2` (SSH implementation)

**Development Dependencies:**
- Testing: `pytest`, `pytest-cov`, `pytest-xdist`, `mock`
- Linting: `flake8` (max-complexity: 10)
- Documentation: `sphinx`, `sphinxcontrib-napoleon`
- Build tools: `tox`, `twine`, `check-manifest`

### Testing

#### Test Structure
```
tests/
├── __init__.py
├── test_forwarder.py     # Main test suite (~1600 lines)
├── requirements.txt       # Test dependencies
├── requirements-syntax.txt # Syntax check dependencies
├── testconfig            # Test SSH config
├── testrsa.key          # Test SSH key
└── testrsa_encrypted.key # Encrypted test key

e2e_tests/
├── run_docker_e2e_db_tests.py     # Database connection tests
├── run_docker_e2e_hangs_tests.py  # Hang prevention tests
└── docker-compose.yaml             # Test infrastructure
```

#### Running Tests

**Using tox (recommended):**
```bash
tox                    # Run all environments
tox -e py38            # Run specific Python version
tox -e syntax          # Run syntax checks
tox -e docs            # Build documentation
```

**Using pytest directly:**
```bash
pytest tests --showlocals --cov sshtunnel --cov-report=term -n4
```

**E2E database tests:**
```bash
cd e2e_tests && docker-compose up -d
python e2e_tests/run_docker_e2e_db_tests.py
python e2e_tests/run_docker_e2e_hangs_tests.py
```

#### Test Configuration
- Tests use `mock` extensively for SSH connections
- Tests are parallelized with `-n4` (pytest-xdist)
- Coverage reporting enabled
- Deprecation warnings ignored during tests

### Code Quality Standards

#### PEP8 Compliance
```bash
flake8 --exclude .venv,build,docs,e2e_tests --max-complexity 10 --ignore=W504
```

**Key rules:**
- Max complexity: 10
- Ignore W504 (line break after binary operator)
- Exclude: `.tox`, `*.egg`, `build`, `data`, `docs`

#### Syntax Checks
```bash
check-manifest --ignore "tox.ini,tests*,*.yml"
python setup.py sdist
twine check dist/*
bashtest README.rst  # Validate CLI examples in README
```

### Continuous Integration

#### CircleCI (.circleci/config.yml)
- Matrix testing: Python 2.7, 3.4, 3.5, 3.6, 3.7, 3.8
- Jobs: `tests`, `docs`, `syntax`, `testdeploy`, `deploy`
- Uses `pipenv` for dependency management
- Coveralls integration for coverage reporting
- Artifact storage for test results and documentation

#### GitHub Actions (.github/workflows/database.yml)
- Database integration tests (Postgres, MySQL, MongoDB)
- Hang prevention tests (with timeout)
- Docker-based end-to-end testing
- Runs on Ubuntu 20.04 with Python 2 and Python 3

#### AppVeyor (appveyor.yml)
- Windows compatibility testing

### Documentation

Located in `docs/` directory:
- Built with Sphinx
- Napoleon extension for Google/NumPy style docstrings
- Available at ReadTheDocs: http://sshtunnel.readthedocs.io/

Build documentation:
```bash
cd docs
sphinx-build -WavE -b html . _build/html
```

## Key Conventions and Guidelines

### Threading Considerations

The library uses threading extensively. Critical points:

1. **Daemon Threads** (line 50)
   - `_DAEMON = True` by default
   - Tunnel threads are daemonized to prevent hangs
   - Changed in v0.4.0 for better hang prevention

2. **Thread Safety**
   - `SSHTunnelForwarder` uses internal locks
   - A deadlock issue was addressed in recent versions (#231)
   - Be careful when modifying lock-related code

3. **Transport Timeout**
   - `SSH_TIMEOUT = 0.1` seconds (line 46)
   - `TUNNEL_TIMEOUT = 10.0` seconds (line 48)
   - These values were tuned to prevent blocking

### Context Manager Behavior

As of v0.3.0, the context manager uses `.stop(force=True)` on exit:
```python
with SSHTunnelForwarder(...) as server:
    # use server
# Automatically calls server.stop(force=True)
```

This is **not fully backward compatible** with pre-0.3.0 versions.

### Deprecation Handling

The library handles deprecated parameters (line 52-57):
```python
_DEPRECATIONS = {
    'ssh_address': 'ssh_address_or_host',
    'ssh_host': 'ssh_address_or_host',
    'ssh_private_key': 'ssh_pkey',
    'raise_exception_if_any_forwarder_have_a_problem': 'mute_exceptions'
}
```

When adding new parameters, maintain backward compatibility and use warnings.

### Logging

Custom logging setup with TRACE level (line 61-62):
```python
TRACE_LEVEL = 1
logging.addLevelName(TRACE_LEVEL, 'TRACE')
```

Logger naming convention: `'sshtunnel.SSHTunnelForwarder'`

### Security Considerations

1. **SSH Host Key Verification**
   - Support for host key checking
   - Configurable SSH config file parsing
   - Host key directories: `~/.ssh/` by default

2. **Authentication Methods**
   - Password authentication
   - Private key files (RSA/ECDSA/Ed25519)
   - SSH agent support (can be disabled)
   - Password-protected private keys

3. **Compression**
   - Optional SSH compression with `-z` flag

### Common Pitfalls and Issues

1. **Hangs and Deadlocks**
   - Major focus in v0.3.0 and recent development
   - Issues #173, #201, #162, #211, #231 addressed this
   - Transport keepalive set to 5 seconds by default
   - Daemon threads prevent hanging on exit

2. **KeyboardInterrupt Handling**
   - Recent fix (#296) to re-raise KeyboardInterrupt in `__enter__`
   - Important for graceful CLI shutdown

3. **Windows Compatibility**
   - No UNIX domain socket support on Windows
   - AppVeyor CI ensures Windows compatibility
   - Check `os.name != 'posix'` for platform-specific code

4. **Bulk Transfer Performance**
   - Optimizations in #247 for speed improvements
   - Relevant for large data transfers through tunnel

## Making Changes

### Before You Start

1. **Read the changelog** (`changelog.rst`) to understand recent changes
2. **Check existing issues** for related work or known problems
3. **Review recent commits** for context on the codebase state

### Code Modifications

1. **Maintain Compatibility**
   - Python 2.7 and 3.4-3.8 support required
   - Use existing compatibility patterns
   - Add deprecation warnings for breaking changes

2. **Testing is Mandatory**
   - Add tests for new features in `tests/test_forwarder.py`
   - Ensure all existing tests pass: `tox`
   - Run syntax checks: `tox -e syntax`
   - For tunnel behavior changes, add e2e tests

3. **Documentation**
   - Update docstrings (Google/NumPy style)
   - Update `README.rst` for user-facing changes
   - Update `docs.rst` for API changes

4. **Logging**
   - Don't modify user-provided loggers (#250)
   - Use appropriate log levels
   - Maintain the `self.info` pattern for debugging

5. **Version Updates**
   - Version is in `sshtunnel.py` line 41: `__version__ = '0.4.0'`
   - Update `changelog.rst` with changes and contributor credits

### Common Tasks

#### Adding a New Parameter to SSHTunnelForwarder

1. Add to `__init__` method signature
2. Document in docstring
3. Store as instance variable
4. Add validation if needed
5. Update tests
6. Document in README if user-facing

#### Modifying Threading Behavior

1. **Exercise extreme caution** - threading bugs are hard to debug
2. Review recent hang-related issues and PRs
3. Add e2e hang tests in `e2e_tests/run_docker_e2e_hangs_tests.py`
4. Test on multiple platforms (Linux, macOS, Windows)
5. Consider impact on daemon thread behavior

#### Adding a New Authentication Method

1. Update `_get_transport` method in `SSHTunnelForwarder`
2. Add CLI argument in `_parse_arguments`
3. Add test with mock SSH connection
4. Document in README with example
5. Update API documentation

### Pull Request Checklist

- [ ] All tests pass: `tox`
- [ ] Syntax checks pass: `tox -e syntax`
- [ ] Documentation builds: `tox -e docs`
- [ ] README examples tested: `bashtest README.rst`
- [ ] Python 2.7 compatibility verified
- [ ] Windows compatibility considered
- [ ] Changelog updated with change and credit
- [ ] Docstrings updated
- [ ] No new flake8 warnings

## Debugging Tips

### Enable Verbose Logging

```python
import logging
logger = logging.getLogger('sshtunnel')
logger.setLevel(logging.DEBUG)
logger.addHandler(logging.StreamHandler())
```

Or use CLI: `sshtunnel -v` (ERROR), `-vv` (WARNING), `-vvv` (INFO), `-vvvv` (DEBUG)

### Common Debug Patterns

1. **Connection Issues**: Check `self.info` attribute on SSHTunnelForwarder
2. **Threading Issues**: Add `TRACE` level logging
3. **Transport Issues**: Enable paramiko transport logging
4. **Port Binding**: Check `local_bind_port` and `local_bind_address`

### Using the Test Infrastructure

```python
# Mock SSH connection for testing
from tests.test_forwarder import open_tunnel
# This partial has safe defaults for testing

with open_tunnel(...) as tunnel:
    # Test code
```

## Additional Resources

- **Troubleshooting**: See `Troubleshoot.rst`
- **Examples**: See `README.rst` Examples 1-4
- **API Reference**: Generated docs at ReadTheDocs
- **Paramiko Documentation**: http://www.paramiko.org/
- **Original Inspiration**: https://github.com/jmagnusson/bgtunnel
- **Paramiko Forward Demo**: https://github.com/paramiko/paramiko/blob/master/demos/forward.py

## Release Process

1. Update version in `sshtunnel.py` (`__version__`)
2. Update `changelog.rst` with all changes and contributors
3. Run full test suite: `tox`
4. Build distributions: `python setup.py bdist_egg bdist_wheel sdist`
5. Check distributions: `twine check dist/*`
6. Upload to TestPyPI: `twine upload --repository testpypi dist/*`
7. Test installation from TestPyPI
8. Upload to PyPI: `twine upload dist/*` (requires token)

Note: CircleCI automates deployment from master branch with manual approval.

---

**Last Updated**: 2025-11-18
**Version**: 0.4.0+
**Maintainer**: Pahaz White

When in doubt, consult the existing code patterns, test suite, and recent issues/PRs for guidance.
