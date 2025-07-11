# We cannot use $(shell pwd), which will return unix path format on Windows,
# making it hard to use.
cur_dir = $(dir $(abspath $(firstword $(MAKEFILE_LIST))))

TOP := $(cur_dir)
# RUSTFLAGS that are likely to be tweaked by developers. For example,
# while we enable debug logs by default here, some might want to strip them
# for minimal code size / consumed cycles.
CUSTOM_RUSTFLAGS := -C debug-assertions
# RUSTFLAGS that are less likely to be tweaked by developers. Most likely
# one would want to keep the default values here.
FULL_RUSTFLAGS := -C target-feature=+zba,+zbb,+zbc,+zbs,-a $(CUSTOM_RUSTFLAGS)
# Additional cargo args to append here. For example, one can use
# make test CARGO_ARGS="-- --nocapture" so as to inspect data emitted to
# stdout in unit tests
CARGO_ARGS :=
MODE := release
# Tweak this to change the clang version to use for building C code. By default
# we use a bash script with somes heuristics to find clang in current system.
CLANG := $(shell $(TOP)/scripts/find_clang)
CLANG_OPTIMIZER := -O2
AR := $(subst clang,llvm-ar,$(CLANG))
OBJCOPY := $(subst clang,llvm-objcopy,$(CLANG))
CLANG_FORMAT := $(subst clang,clang-format,$(CLANG))
# When this is set to some value, the generated binaries will be copied over
BUILD_DIR :=
# Generated binaries to copy. By convention, a Rust crate's directory name will
# likely match the crate name, which is also the name of the final binary.
# However if this is not the case, you can tweak this variable. As the name hints,
# more than one binary is supported here.
BINARIES := $(notdir $(shell pwd))

ifeq (release,$(MODE))
	MODE_ARGS := --release
endif

default: build test

CFLAGS_CKBVM_BASE = --target=riscv64 -march=rv64imc_zba_zbb_zbc_zbs $(CLANG_OPTIMIZER) -fdata-sections -ffunction-sections
CFLAGS_CKBVM_SECP256K1 = $(CFLAGS_CKBVM_BASE) \
	-isystem musl/release/include \
	-DECMULT_WINDOW_SIZE=6 -DENABLE_MODULE_RECOVERY -DENABLE_MODULE_SCHNORRSIG -DENABLE_MODULE_EXTRAKEYS
CFLAGS_CKBVM_SECP256K1_SCHNORR = $(CFLAGS_CKBVM_BASE) \
	-isystem musl/release/include \
	-Isecp256k1/include -Isecp256k1/src \
	-Lmusl/release/lib -Lcompiler-rt-builtins-riscv/build \
	-lc -lgcc -lcompiler-rt \
	-Wl,--gc-sections

CFLAGS_NATIVE_BASE = $(CLANG_OPTIMIZER) -fdata-sections -ffunction-sections
CFLAGS_NATIVE_SECP256K1 = $(CFLAGS_NATIVE_BASE) \
	-DECMULT_WINDOW_SIZE=6 -DENABLE_MODULE_RECOVERY -DENABLE_MODULE_SCHNORRSIG -DENABLE_MODULE_EXTRAKEYS
CFLAGS_NATIVE_SECP256K1_SCHNORR = $(CFLAGS_NATIVE_BASE) \
	-Isecp256k1/include -Isecp256k1/src \
	-Wl,--gc-sections

build: compiler-rt-builtins-riscv musl secp256k1
	$(CLANG) $(CFLAGS_CKBVM_SECP256K1_SCHNORR) -o secp256k1_schnorr_ckbvm secp256k1_schnorr.c secp256k1/build/ckbvm/*
	$(CLANG) $(CFLAGS_NATIVE_SECP256K1_SCHNORR) -o secp256k1_schnorr_native secp256k1_schnorr.c secp256k1/build/native/*
	@set -eu; \
	if [ "x$(BUILD_DIR)" != "x" ]; then \
		for binary in $(BINARIES); do \
			echo "Copying binary $${binary}_ckbvm to build directory"; \
			cp $(TOP)/contracts/$$binary/$${binary}_ckbvm $(TOP)/$(BUILD_DIR); \
			cp $(TOP)/$(BUILD_DIR)/$${binary}_ckbvm $(TOP)/$(BUILD_DIR)/$${binary}_ckbvm.debug; \
			$(OBJCOPY) --strip-debug --strip-all $(TOP)/$(BUILD_DIR)/$${binary}_ckbvm; \
			echo "Copying binary $${binary}_native to build directory"; \
			cp $(TOP)/contracts/$$binary/$${binary}_native $(TOP)/$(BUILD_DIR); \
		done; \
	fi

compiler-rt-builtins-riscv:
	ln -s ../../../../deps/compiler-rt-builtins-riscv .
	cd compiler-rt-builtins-riscv && make CC=$(CLANG) AR=$(AR)

musl:
	ln -s ../../../../deps/musl .
	cd musl && ./ckb/build.sh

secp256k1:
	ln -s ../../../../deps/secp256k1 .
	mkdir -p secp256k1/build
	mkdir -p secp256k1/build/ckbvm
	mkdir -p secp256k1/build/native
	$(CLANG) $(CFLAGS_CKBVM_SECP256K1) -c -o secp256k1/build/ckbvm/precomputed_ecmult.o secp256k1/src/precomputed_ecmult.c
	$(CLANG) $(CFLAGS_CKBVM_SECP256K1) -c -o secp256k1/build/ckbvm/precomputed_ecmult_gen.o secp256k1/src/precomputed_ecmult_gen.c
	$(CLANG) $(CFLAGS_CKBVM_SECP256K1) -c -o secp256k1/build/ckbvm/secp256k1.o secp256k1/src/secp256k1.c
	$(CLANG) $(CFLAGS_NATIVE_SECP256K1) -c -o secp256k1/build/native/precomputed_ecmult.o secp256k1/src/precomputed_ecmult.c
	$(CLANG) $(CFLAGS_NATIVE_SECP256K1) -c -o secp256k1/build/native/precomputed_ecmult_gen.o secp256k1/src/precomputed_ecmult_gen.c
	$(CLANG) $(CFLAGS_NATIVE_SECP256K1) -c -o secp256k1/build/native/secp256k1.o secp256k1/src/secp256k1.c

# test, check, clippy and fmt here are provided for completeness,
# there is nothing wrong invoking cargo directly instead of make.
test:
	# cargo test $(CARGO_ARGS)
	echo

check:
	cargo check $(CARGO_ARGS)

clippy:
	cargo clippy $(CARGO_ARGS)

STYLE := "{BasedOnStyle: Google, TabWidth: 4, IndentWidth: 4, UseTab: Never, SortIncludes: false, ColumnLimit: 120}"
fmt:
	$(CLANG_FORMAT) -i -style=$(STYLE) *.c

# Arbitrary cargo command is supported here. For example:
#
# make cargo CARGO_CMD=expand CARGO_ARGS="--ugly"
#
# Invokes:
# cargo expand --ugly
CARGO_CMD :=
cargo:
	cargo $(CARGO_CMD) $(CARGO_ARGS)

clean:
	rm -f $(BINARIES)

deps-clean:
	cd compiler-rt-builtins-riscv && make clean || true
	cd musl && make clean || true
	rm -rf secp256k1/build || true
	rm -rf compiler-rt-builtins-riscv musl secp256k1 || true

prepare:
	rustup target add riscv64imac-unknown-none-elf

.PHONY: build test check clippy fmt cargo clean prepare
