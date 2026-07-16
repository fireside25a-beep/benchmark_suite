#!/usr/bin/env bash
set -euo pipefail

# Build a small deterministic mathematics benchmark suite.
# Existing ./benchmarks content is backed up before replacement.

if command -v python3 >/dev/null 2>&1; then
    PYTHON_BIN="python3"
elif command -v python >/dev/null 2>&1; then
    PYTHON_BIN="python"
else
    echo "ERROR: Python 3 was not found in PATH." >&2
    exit 1
fi

if [ -d "benchmarks" ]; then
    backup_dir="benchmarks_backup_$(date +%Y%m%d_%H%M%S)"
    cp -R benchmarks "$backup_dir"
    echo "Backed up existing benchmarks directory to: $backup_dir"
fi

mkdir -p benchmarks/modules benchmarks/tests

cat <<'EOF' > benchmarks/__init__.py
"""Deterministic benchmark suite package."""
EOF

cat <<'EOF' > benchmarks/config.json
{
  "suite_name": "Deterministic Mathematics Verification",
  "version": "1.0.0",
  "parameters": {
    "stress_iterations": 1000
  }
}
EOF

cat <<'EOF' > benchmarks/modules/__init__.py
"""Reusable benchmark modules."""
EOF

cat <<'EOF' > benchmarks/modules/determinism.py
"""Deterministic matrix workloads and exact reference checks."""

from typing import List, Sequence, Tuple

Matrix = List[List[int]]
ReadOnlyMatrix = Sequence[Sequence[int]]


def _validate_matrix(matrix: ReadOnlyMatrix, name: str) -> Tuple[int, int]:
    if not matrix or not matrix[0]:
        raise ValueError(f"{name} must be a non-empty matrix")

    column_count = len(matrix[0])
    if any(len(row) != column_count for row in matrix):
        raise ValueError(f"{name} must be rectangular")

    return len(matrix), column_count


def multiply_matrices(a: ReadOnlyMatrix, b: ReadOnlyMatrix) -> Matrix:
    """Multiply two compatible integer matrices using pure Python."""
    a_rows, a_columns = _validate_matrix(a, "a")
    b_rows, b_columns = _validate_matrix(b, "b")

    if a_columns != b_rows:
        raise ValueError(
            "Incompatible matrix dimensions: "
            f"a is {a_rows}x{a_columns}, b is {b_rows}x{b_columns}"
        )

    return [
        [
            sum(a[row][index] * b[index][column] for index in range(a_columns))
            for column in range(b_columns)
        ]
        for row in range(a_rows)
    ]


def get_matrix_2x2() -> Tuple[Matrix, Matrix]:
    return [[1, 2], [3, 4]], [[5, 6], [7, 8]]


def get_matrix_4x4() -> Tuple[Matrix, Matrix]:
    a = [
        [1, 0, 0, 1],
        [0, 2, 1, 0],
        [1, 0, 2, 1],
        [0, 1, 1, 3],
    ]
    b = [
        [4, 1, 1, 0],
        [1, 2, 0, 1],
        [1, 1, 2, 1],
        [0, 1, 1, 3],
    ]
    return a, b


def verify_product_2x2(product: ReadOnlyMatrix) -> Tuple[bool, Matrix]:
    expected = [[19, 22], [43, 50]]
    return list(map(list, product)) == expected, expected


def verify_product_4x4(product: ReadOnlyMatrix) -> Tuple[bool, Matrix]:
    expected = [
        [4, 2, 2, 3],
        [3, 5, 2, 3],
        [6, 4, 6, 5],
        [2, 6, 5, 11],
    ]
    return list(map(list, product)) == expected, expected
EOF

cat <<'EOF' > benchmarks/modules/latency.py
"""Deterministic arithmetic workload used for timing and verification."""


def generate_algebraic_bounds(iterations: int = 1000) -> int:
    """Run the benchmark workload using an explicit loop."""
    if not isinstance(iterations, int) or isinstance(iterations, bool):
        raise TypeError("iterations must be an integer")
    if iterations < 1:
        raise ValueError("iterations must be at least 1")

    total_sum = 0
    for i in range(1, iterations + 1):
        total_sum += ((i * 3) + 7) * ((i * 5) + 11)
    return total_sum


def expected_algebraic_bounds(iterations: int = 1000) -> int:
    """Calculate the same target independently with closed-form sums."""
    if not isinstance(iterations, int) or isinstance(iterations, bool):
        raise TypeError("iterations must be an integer")
    if iterations < 1:
        raise ValueError("iterations must be at least 1")

    sum_i = iterations * (iterations + 1) // 2
    sum_i_squared = iterations * (iterations + 1) * (2 * iterations + 1) // 6

    # ((3i + 7) * (5i + 11)) = 15i^2 + 68i + 77
    return (15 * sum_i_squared) + (68 * sum_i) + (77 * iterations)
EOF

cat <<'EOF' > benchmarks/run_suite.py
"""Command-line runner for the deterministic benchmark suite."""

import json
import sys
import time
from pathlib import Path
from typing import Any, Dict, Tuple

if __package__:
    from .modules.determinism import (
        get_matrix_2x2,
        get_matrix_4x4,
        multiply_matrices,
        verify_product_2x2,
        verify_product_4x4,
    )
    from .modules.latency import (
        expected_algebraic_bounds,
        generate_algebraic_bounds,
    )
else:
    from modules.determinism import (
        get_matrix_2x2,
        get_matrix_4x4,
        multiply_matrices,
        verify_product_2x2,
        verify_product_4x4,
    )
    from modules.latency import (
        expected_algebraic_bounds,
        generate_algebraic_bounds,
    )

CONFIG_PATH = Path(__file__).with_name("config.json")


def load_config() -> Dict[str, Any]:
    try:
        with CONFIG_PATH.open("r", encoding="utf-8") as config_file:
            config = json.load(config_file)
    except FileNotFoundError as exc:
        raise RuntimeError(f"Missing configuration file: {CONFIG_PATH}") from exc
    except json.JSONDecodeError as exc:
        raise RuntimeError(f"Invalid JSON in {CONFIG_PATH}: {exc}") from exc

    parameters = config.get("parameters")
    if not isinstance(parameters, dict):
        raise RuntimeError("config.json must contain a 'parameters' object")

    iterations = parameters.get("stress_iterations")
    if not isinstance(iterations, int) or isinstance(iterations, bool) or iterations < 1:
        raise RuntimeError("stress_iterations must be a positive integer")

    return config


def run_matrix_case(size: str) -> Tuple[bool, float]:
    if size == "2x2":
        a, b = get_matrix_2x2()
        verifier = verify_product_2x2
    elif size == "4x4":
        a, b = get_matrix_4x4()
        verifier = verify_product_4x4
    else:
        raise ValueError(f"Unsupported matrix case: {size}")

    start_ns = time.perf_counter_ns()
    product = multiply_matrices(a, b)
    end_ns = time.perf_counter_ns()

    passed, expected = verifier(product)
    duration_us = (end_ns - start_ns) / 1000.0

    print(f"{size} Match: {passed} | Duration: {duration_us:.3f} us")
    if not passed:
        print(f"  Actual:   {product}")
        print(f"  Expected: {expected}")

    return passed, duration_us


def execute_benchmark() -> int:
    config = load_config()
    suite_name = config.get("suite_name", "Unnamed Suite")
    version = config.get("version", "unknown")
    iterations = config["parameters"]["stress_iterations"]

    print("=" * 62)
    print(f"{suite_name} | Version {version}")
    print("=" * 62)

    results = []

    passed_2x2, _ = run_matrix_case("2x2")
    results.append(("2x2 matrix product", passed_2x2))

    passed_4x4, _ = run_matrix_case("4x4")
    results.append(("4x4 matrix product", passed_4x4))

    print(f"\nEvaluating {iterations}-row deterministic arithmetic workload...")
    start_ns = time.perf_counter_ns()
    actual_scalar = generate_algebraic_bounds(iterations)
    end_ns = time.perf_counter_ns()
    expected_scalar = expected_algebraic_bounds(iterations)
    scalar_passed = actual_scalar == expected_scalar
    scalar_duration_us = (end_ns - start_ns) / 1000.0

    print(f"Scalar Match: {scalar_passed} | Duration: {scalar_duration_us:.3f} us")
    print(f"Scalar Result Z: {actual_scalar}")
    results.append(("algebraic workload", scalar_passed))

    passed_count = sum(1 for _, passed in results if passed)
    total_count = len(results)

    print("\n" + "-" * 62)
    for test_name, passed in results:
        status = "PASS" if passed else "FAIL"
        print(f"[{status}] {test_name}")
    print("-" * 62)
    print(f"Suite Result: {passed_count}/{total_count} checks passed")

    if passed_count == total_count:
        print("=== SUITE EXECUTION PASSED ===")
        return 0

    print("=== SUITE EXECUTION FAILED ===")
    return 1


if __name__ == "__main__":
    try:
        raise SystemExit(execute_benchmark())
    except (RuntimeError, TypeError, ValueError) as exc:
        print(f"ERROR: {exc}", file=sys.stderr)
        raise SystemExit(2) from exc
EOF

cat <<'EOF' > benchmarks/tests/__init__.py
"""Unit tests for the deterministic benchmark suite."""
EOF

cat <<'EOF' > benchmarks/tests/test_suite.py
"""Smoke-level correctness tests for all deterministic workloads."""

import unittest

from benchmarks.modules.determinism import (
    get_matrix_2x2,
    get_matrix_4x4,
    multiply_matrices,
    verify_product_2x2,
    verify_product_4x4,
)
from benchmarks.modules.latency import (
    expected_algebraic_bounds,
    generate_algebraic_bounds,
)


class DeterministicSuiteTests(unittest.TestCase):
    def test_2x2_product(self) -> None:
        a, b = get_matrix_2x2()
        product = multiply_matrices(a, b)
        passed, expected = verify_product_2x2(product)
        self.assertTrue(passed)
        self.assertEqual(product, expected)

    def test_4x4_product(self) -> None:
        a, b = get_matrix_4x4()
        product = multiply_matrices(a, b)
        passed, expected = verify_product_4x4(product)
        self.assertTrue(passed)
        self.assertEqual(product, expected)

    def test_1000_iteration_scalar(self) -> None:
        actual = generate_algebraic_bounds(1000)
        expected = expected_algebraic_bounds(1000)
        self.assertEqual(actual, 5_041_613_500)
        self.assertEqual(actual, expected)

    def test_invalid_dimensions_are_rejected(self) -> None:
        with self.assertRaises(ValueError):
            multiply_matrices([[1, 2]], [[1, 2]])


if __name__ == "__main__":
    unittest.main()
EOF

echo
echo "Running syntax compilation..."
"$PYTHON_BIN" -m py_compile \
    benchmarks/__init__.py \
    benchmarks/modules/__init__.py \
    benchmarks/modules/determinism.py \
    benchmarks/modules/latency.py \
    benchmarks/run_suite.py \
    benchmarks/tests/__init__.py \
    benchmarks/tests/test_suite.py

echo "Syntax compilation: PASS"

echo
echo "Running unit smoke tests..."
"$PYTHON_BIN" -m unittest discover -s benchmarks/tests -v

echo
echo "Running complete benchmark suite..."
"$PYTHON_BIN" -m benchmarks.run_suite

echo
echo "Builder completed successfully."
echo "Run again later with: $PYTHON_BIN -m benchmarks.run_suite"
