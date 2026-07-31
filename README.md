# p6m7g8-actions/python-setup

- [p6m7g8-actions/python-setup](#p6m7g8-actionspython-setup)
  - [Usage](#usage)

## Usage

```yml
    - name: Python
      uses: p6m7g8-actions/p6-python-setup@main
      with:
        python-version: 3.14.3
```

This action does not check out the repository; check it out first if you want
project dependencies installed.

With a `uv.lock` in the working directory, `uv` is installed and
`uv sync --dev` installs the project's dependencies. Without one, the action
installs the interpreter only and emits a warning annotation saying so.
