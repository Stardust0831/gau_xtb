# gau_xtb.py: a cross-platform Gaussian-xTB External interface

`gau_xtb.py` is a pure Python interface for coupling Gaussian with Grimme's
xTB program through Gaussian's `External` keyword. Gaussian controls the
optimization, transition-state search, IRC, and vibrational analysis workflow,
while xTB supplies the energy, gradient, and Hessian.

This project is a Python rewrite of the original Linux-oriented `gau_xtb`
workflow. The runtime code uses only the Python standard library, so the same
script can be used on Linux, Windows, and HPC clusters as long as Python,
Gaussian, and xTB are available.

Original interface to cite:

Tian Lu, gau_xtb: A Gaussian interface for xtb code,
http://sobereva.com/soft/gau_xtb (accessed month day, year).

Chinese tutorial:

跨平台实现gaussian与xTB程序联用搜索过渡态、产生IRC、做振动分析  
http://bbs.keinsci.com/forum.php?mod=viewthread&tid=59419&fromuid=58653  
(出处: 计算化学公社)

## Requirements

- Python 3.9 or newer
- Gaussian 09/16 with the `External` keyword
- xTB executable supporting `--sp`, `--grad`, and `--hess`
- No third-party Python runtime packages

The `requirements.txt` file is intentionally empty apart from comments. A
minimal Conda environment is provided in `environment.yml`.

## Basic Usage

Make sure xTB can be found from the same environment that starts Gaussian:

```bash
xtb --version
```

On Linux or macOS, make the script executable:

```bash
chmod +x /path/to/gau_xtb/gau_xtb.py
```

Then use it in a Gaussian route section:

```text
#P opt(ts,calcfc,noeigen,nomicro) external='/path/to/gau_xtb/gau_xtb.py'
```

You can also call it through an explicit Python interpreter, which is often the
most portable form:

```text
#P opt(ts,calcfc,noeigen,nomicro) external='python3 /path/to/gau_xtb/gau_xtb.py'
```

On Windows, use the absolute path to the Python script, for example:

```text
#P opt(ts,calcfc,noeigen,nomicro) external='python3 D:\dft\test\gau_xtb\gau_xtb.py'
```

Here `python3` is only an example. Depending on your Windows installation, use
`python`, `py -3`, or the absolute path to `python.exe`. If the path contains
spaces, quoting can become fragile inside Gaussian input files; using a working
directory without spaces or non-ASCII characters is recommended.

## Frequency Calculations

For Gaussian 16, an optimization followed by a frequency calculation often
works in one route section:

```text
#P opt(ts,calcfc,noeigen,nomicro) freq external='python3 /path/to/gau_xtb/gau_xtb.py'
```

For Gaussian 09 on Windows, the automatic follow-up frequency step may insert
`Guess=TCheck` and fail before the external script is called:

```text
No guess information found on chk file.
```

This is a Gaussian checkpoint/guess issue, not a problem with the Python
interface or xTB output. A safer Gaussian 09W workflow is to use an explicit
Link1 input:

```text
%chk=mol.chk
#P opt(ts,calcfc,noeigen,nomicro) external='python3 D:\dft\test\gau_xtb\gau_xtb.py'

Title

0 1
... geometry ...

--Link1--
%chk=mol.chk
#P freq geom=allcheck external='python3 D:\dft\test\gau_xtb\gau_xtb.py'
```

If the local Gaussian 09W installation still rejects the Link1 form, run the
optimization and frequency route sections as two separate Gaussian jobs while
reusing the same checkpoint file.

## TS, Frequency, and IRC Example

The following input performs a transition-state search, then a frequency
calculation, and finally an IRC calculation. Replace
`D:\work\bbs\gau_xtb\gau_xtb.py` with the real absolute path to this script.
On Linux, use an absolute path such as
`/home/user/work/gau_xtb/gau_xtb.py`; do not rely on `~` expansion inside a
Gaussian `External` command.

```text
%chk=D:\work\bbs\gau_xtb\TS.chk
%nprocshared=1
#P opt=(calcfc,ts,noeigen,nomicro) external='python3 D:\work\bbs\gau_xtb\gau_xtb.py'

Title Card Required

0 1
C                 -1.63584300   -0.21808700    0.18089800
C                 -0.31253400   -0.78768400   -0.12991700
C                 -0.69293200    0.88701800   -0.24226100
H                 -2.44474100   -0.49942200   -0.48067100
H                 -1.93454100   -0.16341300    1.21850400
H                 -0.17138300   -1.46736000   -0.97247900
H                 -0.63822800    1.76626700    0.38321600
H                 -0.66552900    1.13166600   -1.29982000
C                  0.66396300    0.00090100    0.49925300
O                  1.89764600   -0.06104200   -0.13643800
H                  2.53733300    0.42770600    0.39491700

--Link1--
%oldchk=D:\work\bbs\gau_xtb\TS.chk
%chk=D:\work\bbs\gau_xtb\freq.chk
%nprocshared=1
#P freq geom=allcheck external='python3 D:\work\bbs\gau_xtb\gau_xtb.py'

--Link1--
%oldchk=D:\work\bbs\gau_xtb\freq.chk
%chk=D:\work\bbs\gau_xtb\IRC.chk
%nprocshared=1
#P IRC(maxpoints=20,calcfc) geom=allcheck external='python3 D:\work\bbs\gau_xtb\gau_xtb.py'
```

`%oldchk` reads information from the previous checkpoint file, while `%chk`
sets the checkpoint file for the current step. Gaussian is used mainly as the
driver here, so `%nprocshared=1` is usually enough. Control the xTB thread
count with `GAU_XTB_THREADS` instead.

## How It Works

Gaussian calls an `External` program with arguments like:

```text
script layer InputFile OutputFile MsgFile FChkFile MatElFile
```

`gau_xtb.py` mainly uses `InputFile` and `OutputFile`:

- It reads atom numbers, coordinates, charge, multiplicity, and requested
  derivative level from the Gaussian External input file.
- It writes a temporary `mol.xyz` file for xTB.
- It runs xTB and parses `xtbout`, `gradient`, and `hessian`.
- It writes energy, dipole placeholder, gradient, and Hessian back in the
  Gaussian External output format.

The derivative level requested by Gaussian controls the xTB command:

```text
Derivs = 0    xtb mol.xyz --sp
Derivs = 1    xtb mol.xyz --grad
Derivs = 2    xtb mol.xyz --hess --grad
```

## Configuration

Configuration is done through environment variables:

```bash
export GAU_XTB_COMMAND=xtb
export GAU_XTB_THREADS=8
export GAU_XTB_GFN=2
export GAU_XTB_ARGS="--alpb water"
export GAU_XTB_TMPDIR=/tmp
export GAU_XTB_TIMEOUT=600
```

Meaning:

- `GAU_XTB_COMMAND`: xTB executable path or command. Default: `xtb`.
- `GAU_XTB_THREADS`: sets both `OMP_NUM_THREADS` and `MKL_NUM_THREADS`.
- `GAU_XTB_GFN`: appends `--gfn VALUE`, for example `2`.
- `GAU_XTB_ARGS`: extra xTB arguments, parsed like a command line.
- `GAU_XTB_TMPDIR`: parent directory for per-call temporary directories.
- `GAU_XTB_TIMEOUT`: xTB timeout in seconds.

For Slurm jobs, prefer one MPI task with multiple CPU threads:

```bash
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=16

export GAU_XTB_THREADS=$SLURM_CPUS_PER_TASK
g16 < TS.gjf > TS.log
```

On Windows `cmd.exe`, the same variables can be set before launching Gaussian:

```bat
set GAU_XTB_COMMAND=xtb
set GAU_XTB_THREADS=8
set GAU_XTB_GFN=2
set GAU_XTB_ARGS=--alpb water
```

## Helper Commands

The script can also be used directly for testing and format conversion:

```bash
python3 -m gau_xtb.gau_xtb genxyz InputFile mol.xyz
python3 -m gau_xtb.gau_xtb extderi OutputFile NAtoms Derivs --workdir DIR
python3 -m gau_xtb.gau_xtb run R InputFile OutputFile
```

## Tests

Run the unit tests with:

```bash
python3 -m unittest discover -s gau_xtb/tests -v
```

## Notes

- The Python interface does not replace Gaussian or xTB. Both external
  programs must be installed separately.
- `External=(...,NoGuess)` is a Gaussian 16 External option and should not be
  treated as a portable Gaussian 09W workaround.
- xTB is used here as a fast semiempirical method for generating approximate
  reaction-path information. Final energies and mechanisms should be checked at
  an appropriate higher level of theory.

## Citation

If this Python interface is useful in your work, please mention this project or
the Chinese tutorial above. Please also cite the original interface:

Tian Lu, gau_xtb: A Gaussian interface for xtb code,
http://sobereva.com/soft/gau_xtb (accessed month day, year).
