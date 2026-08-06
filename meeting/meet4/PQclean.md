# NIST Standards Verification Report for the ML-KEM Implementation Used in PQClean

---

# 1. Purpose of the Report

## 1.1 Objective

This report is prepared to verify whether the post-quantum cryptographic algorithm implemented by the PQClean library corresponds to the standardized Module-Lattice-Based Key Encapsulation Mechanism (ML-KEM) specified by the National Institute of Standards and Technology (NIST).

The objective of this assessment is to:

- Identify the cryptographic algorithm implemented by the PQClean library.
- Verify whether the algorithm is standardized by NIST.
- Confirm that the ML-KEM-512 implementation corresponds to the specifications defined in NIST FIPS 203.
- Verify the correctness of the implementation using the official PQClean verification suite.
- Verify that the benchmark application integrates and executes the official PQClean ML-KEM-512 implementation.
- Document the verification and validation results obtained during the implementation assessment.

---

# 2. NIST Standard Information

| Specification | Details |
|---------------|---------|
| Standard Name | NIST FIPS 203 |
| Standard Title | Module-Lattice-Based Key-Encapsulation Mechanism (ML-KEM) |
| Standardized Algorithm | ML-KEM |
| Parameter Set Verified | ML-KEM-512 |
| Library Evaluated | PQClean |
| Standardization Authority | National Institute of Standards and Technology (NIST) |
| Verification Basis | Algorithm standardized in NIST FIPS 203 |
| Publication Date | August 13, 2024 |

---

# 3. Verification of the ML-KEM Implementation

The PQClean library was examined to identify the post-quantum cryptographic algorithm implemented by the library.

The official PQClean repository provides clean reference implementations of NIST post-quantum cryptographic algorithms. For this project, the ML-KEM-512 implementation available in the official PQClean repository was selected for evaluation.

The implementation source, repository structure, documentation, and implementation files were examined to verify that the implementation corresponds to the Module-Lattice-Based Key Encapsulation Mechanism (ML-KEM) standardized in NIST FIPS 203.

The implementation was further verified by executing the official PQClean verification suite and validating its successful integration into the benchmark application used in this project.

Based on the implementation analysis and verification results, the PQClean ML-KEM-512 implementation corresponds to the standardized ML-KEM algorithm defined in NIST FIPS 203.

---

# 4. Verification Methodology

The verification process consisted of the following activities:

1. Reviewing the official NIST FIPS 203 specification for ML-KEM.

2. Reviewing the official PQClean documentation and repository.

3. Identifying the ML-KEM-512 implementation used in the project.

4. Recording the implementation revision using the Git commit hash.

5. Executing the official PQClean verification suite.

6. Verifying that the benchmark application invokes the official PQClean ML-KEM-512 implementation.

7. Building and executing the benchmark implementation.

8. Reviewing the verification results to confirm conformity with the standardized ML-KEM specification.

---

## 4.1 Verification Procedure

The NIST standards verification was performed using the following procedure.

### Step 1 – Review of the NIST Standard

The official NIST FIPS 203 specification was reviewed to identify the standardized Module-Lattice-Based Key Encapsulation Mechanism (ML-KEM) and its supported parameter sets.

The ML-KEM-512 parameter set used throughout this project was verified to be one of the parameter sets defined in the standard.


---

### Step 2 – Identification of the PQClean Implementation

The official PQClean repository was cloned from GitHub.

The ML-KEM-512 implementation located under

```
crypto_kem/ml-kem-512/clean
```

was identified as the implementation used for verification.

The exact implementation revision was recorded using the Git commit hash.

Commands executed:

```bash
git clone https://github.com/PQClean/PQClean.git

cd PQClean/crypto_kem/ml-kem-512/clean

git rev-parse HEAD
```

The recorded Git commit uniquely identifies the implementation version evaluated during this verification.

---

### Step 3 – Preparation of the Verification Environment

The official PQClean verification framework located under the `test` directory was used for implementation verification.

A Python virtual environment was created to isolate the verification environment.

The required verification dependencies were installed using the requirements file provided by the PQClean project.

Commands executed:

```bash
cd PQClean/test

python3 -m venv .venv

source .venv/bin/activate

pip install -r ../requirements.txt
```

The successful installation of all required dependencies prepared the verification environment for executing the official PQClean verification suite.


---

### Step 4 – Execution of the Official PQClean Verification Suite

The official PQClean verification suite was executed to verify the correctness of the ML-KEM-512 implementation.

The verification consisted of three independent tests:

- Compile Verification
- Functional Verification
- NIST Known Answer Test (KAT)

Each verification step is documented in the following section.

---
## 5. Algorithm Verification Summary

The verification process confirmed that the cryptographic algorithm implemented by the PQClean library corresponds to the Module-Lattice-Based Key-Encapsulation Mechanism (ML-KEM) standardized by the National Institute of Standards and Technology (NIST) in FIPS 203.

The implementation supports the ML-KEM-512 parameter set evaluated in this project. The verification established that the algorithm implemented by the PQClean library corresponds to the ML-KEM specification defined in NIST FIPS 203.

The implementation was further verified using the official PQClean verification suite. The successful execution of the compile verification, functional verification, and NIST Known Answer Test (KAT) demonstrates that the implementation satisfies the correctness checks provided by the PQClean project.

In addition, source-code verification confirmed that the benchmark implementation directly invokes the official PQClean ML-KEM-512 implementation, and successful benchmark execution demonstrated correct integration of the implementation into the project.

---

## 6. Findings

The verification produced the following findings:

- The PQClean library implements the ML-KEM algorithm standardized by NIST in FIPS 203.
- The ML-KEM-512 parameter set evaluated in this project is defined in NIST FIPS 203.
- The cryptographic algorithm implemented by PQClean corresponds to the standardized ML-KEM specification.
- The official PQClean verification suite completed successfully for the ML-KEM-512 implementation.
- Source-code inspection confirmed that the benchmark directly invokes the official PQClean ML-KEM-512 implementation.
- The benchmark application was successfully compiled and executed using the verified implementation.

---

# 7. Implementation Verification Using the Official PQClean Test Suite

To verify that the ML-KEM implementation used in this project corresponds to the standardized ML-KEM specification defined in NIST FIPS 203, implementation-level verification was performed using the official verification framework provided by the PQClean project.

The verification process consisted of source verification, compile verification, functional verification, NIST Known Answer Test (KAT) verification, source-code inspection, and benchmark execution.

---

## 7.1 Implementation Files Used

The ML-KEM implementation used in this project consists of the following implementation files:

- `api.h`
- `kem.c`
- `kem.h`
- `indcpa.c`
- `poly.c`
- `polyvec.c`
- `ntt.c`
- `reduce.c`
- `verify.c`
- `cbd.c`
- `symmetric-shake.c`
- `fips202.c`

These files correspond to the official PQClean ML-KEM-512 clean implementation located under:

```text
crypto_kem/ml-kem-512/clean
```

The benchmark application uses these implementation files for ML-KEM-512 key generation.


---

## 7.2 Implementation Verification Steps

The following verification procedure was performed.

### Step 1 – Clone the Official PQClean Repository

The official PQClean repository was cloned from GitHub.

```bash
git clone https://github.com/PQClean/PQClean.git
```

The implementation revision was recorded using the Git commit hash.

```bash
git rev-parse HEAD
```
![Relative](./imagesmeet4/Fig%201.png)
**Figure 1.** Cloning the official PQClean repository and recording the repository revision.

---

### Step 2 – Prepare the Verification Environment

The official PQClean verification environment was prepared by creating a Python virtual environment and installing the required dependencies.

```bash
cd PQClean/test

python3 -m venv .venv

source .venv/bin/activate

pip install -r ../requirements.txt
```
![Relative](./imagesmeet4/Fig%202a.png)
**Figure 2(a).** Creating the Verification Environment.

![Relative](./imagesmeet4/Fig%202b.png)

**Figure 2(b).** Successful installation of the required Python packages for the PQClean verification framework. The active virtual environment and successful installation confirm that the verification environment was prepared correctly.

---

### Step 3 – Compile Verification

The compile verification confirms that the ML-KEM-512 implementation can be successfully compiled as a library.

Command executed:

```bash
pytest -v test_compile_lib.py -k "ml-kem-512 and aarch64"
```

Result:

```text
PASSED
```
![Relative](./imagesmeet4/Fig%203.png)
**Figure 3.** Compilation Verification of the ML-KEM-512 Implementation.

---

### Step 4 – Functional Verification

The functional verification confirms that the implementation correctly performs the ML-KEM operations defined by the PQClean verification framework.

Command executed:

```bash
pytest -v test_functest.py -k "ml-kem-512 and aarch64"
```

Result:

```text
PASSED
```

The AddressSanitizer verification is skipped on macOS because the required sanitizer environment is not available.

![Relative](./imagesmeet4/Fig%204.png)
**Figure 4.** Functional Verification of the ML-KEM-512 Implementation.

---

### Step 5 – NIST Known Answer Test (KAT)

The NIST Known Answer Test verifies that the implementation produces the expected deterministic outputs for the supplied test vectors.

Command executed:

```bash
pytest -v test_nistkat.py -k "ml-kem-512 and aarch64"
```

Result:

```text
PASSED
```
![Relative](./imagesmeet4/Fig%205.png)
**Figure 5.** NIST Known Answer Test (KAT) Verification of the ML-KEM-512 Implementation.

   The official PQClean Known Answer Test (test_nistkat.py) was executed for the ML-KEM-512 implementation targeting the aarch64 platform. The test completed successfully, with the selected test case passing without errors. The successful execution of the Known Answer Test indicates that the implementation produces outputs consistent with the reference test vectors used by the PQClean verification framework, providing confidence in the correctness of the ML-KEM-512 implementation.

---

### Step 6 – Verification of Benchmark Integration

The benchmark source code was inspected to verify that the benchmark application invokes the official PQClean ML-KEM-512 implementation for key generation. The verification was performed by locating the ML-KEM-512 key generation API used within the benchmark source code.

Command executed:

```bash
grep -R -n "PQCLEAN_MLKEM512_CLEAN_crypto_kem_keypair" .
```
![Relative](./imagesmeet4/Fig%206.png)
**Figure 6.** Verification that the benchmark invokes the official PQClean ML-KEM-512 key generation function.

The search results confirm that the benchmark invokes the official PQClean key generation function PQCLEAN_MLKEM512_CLEAN_crypto_kem_keypair(). References to this function are present in kem.c, api.h, kem.h, and main.c, demonstrating that the benchmark is directly linked to the official PQClean ML-KEM-512 implementation used for the verification process.

---

### Step 7 – Benchmark Compilation

The benchmark application was compiled using the provided Makefile.

Commands executed:

```bash
make clean

make
```

The compilation completed successfully, producing the benchmark executable.

![Relative](./imagesmeet4/Fig%207.png)

**Figure 7.** Successful benchmark compilation.

---

### Step 8 – Benchmark Execution

The compiled benchmark application was executed.

Command executed:

```bash
make run
```

The key generation benchmark completed successfully and produced timing statistics for the ML-KEM-512 key generation operation.

![Relative](./imagesmeet4/Fig%208.png)
**Figure 8.** Successful execution of the ML-KEM-512 key generation benchmark.

---




# 8. Conclusion

Based on the verification performed against the NIST FIPS 203 standard, the cryptographic algorithm implemented by the PQClean library corresponds to the standardized Module-Lattice-Based Key-Encapsulation Mechanism (ML-KEM) specified by the National Institute of Standards and Technology (NIST).

The evaluation confirmed that the ML-KEM-512 parameter set used in this project is one of the official parameter sets standardized in NIST FIPS 203.

Implementation verification performed using the official PQClean verification suite confirmed that the ML-KEM-512 implementation successfully passed the compile verification, functional verification, and NIST Known Answer Test (KAT). Source-code inspection confirmed that the benchmark application directly invokes the official PQClean implementation, and successful execution of the key generation benchmark demonstrated correct integration of the implementation into the project.

The verification performed in this report provides strong evidence that the implementation used in this project corresponds to the ML-KEM algorithm specified in NIST FIPS 203 for the functionality evaluated.

This report does not constitute official NIST ACVP/CAVP validation or certification of the PQClean software library.

---

# References

1. National Institute of Standards and Technology (NIST), *FIPS 203: Module-Lattice-Based Key-Encapsulation Mechanism (ML-KEM)*, August 13, 2024.

   Available at:

   https://nvlpubs.nist.gov/nistpubs/fips/nist.fips.203.pdf

2. PQClean – Official GitHub Repository and Documentation.

   Available at:

   https://github.com/PQClean/PQClean