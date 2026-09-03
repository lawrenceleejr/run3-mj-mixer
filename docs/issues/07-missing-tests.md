# tests/ referenced by README, the plan doc and pyproject is not in the repository

**Branch:** upstream `full-runs`.

README ("Tests": `python -m pytest tests/ -q`, naming `test_match.py`, `test_index.py`, `test_index_build.py`, `tests/fixtures/oracle/`), `docs/global-hemisphere-index-plan.md` and `pyproject.toml` (`[test]` extra) all describe a test suite, and the matcher's exactness claim ("verified against library.py's brute-force scan for identical partner and bit-identical distance") rests on `test_match.py`. `git ls-tree -r full-runs` contains no `tests/` directory at all. `.gitignore` carries a `test*` pattern with `!tests/` negations; check whether the suite exists locally and was swallowed, and commit it. `main` has the same problem in its README (`pytest`) with no tests.
