# Contributing to ExecuTorch Arduino

Thank you for your interest in contributing to the ExecuTorch Arduino library!
We want to make it easy to contribute to this project.

## Pull Requests

We actively welcome your pull requests.

1. Fork the repo and create your branch from `main`.
2. If you've added code that should be tested, add tests.
3. If you've changed APIs, update the documentation.
4. Ensure your code compiles with `arduino-cli compile`.
5. Make sure your code lints (see below).
6. If you haven't already, complete the Contributor License Agreement ("CLA").

## Contributor License Agreement ("CLA")

In order to accept your pull request, we need you to submit a CLA. You only need
to do this once to work on any of Meta's open source projects.

Complete your CLA here: <https://code.facebook.com/cla>

## Issues

We use [GitHub issues](https://github.com/meta-pytorch/executorch-arduino/issues)
to track public bugs. Please ensure your description is clear and has sufficient
instructions to be able to reproduce the issue.

Meta has a [bounty program](https://www.facebook.com/whitehat/) for the safe
disclosure of security bugs. In those cases, please go through the process
outlined on that page and do not file a public issue.

## Coding Style

* C++ code should follow the
  [Google C++ Style Guide](https://google.github.io/styleguide/cppguide.html)
* Arduino sketches should follow standard Arduino conventions
* Use 2 spaces for indentation in C++ and header files

## Testing

Before submitting a PR, please verify your changes compile:

```bash
arduino-cli compile --fqbn arduino:zephyr:unoq examples/HelloExecuTorch
```

## License

By contributing to ExecuTorch Arduino, you agree that your contributions will be
licensed under the LICENSE file in the root directory of this source tree.
