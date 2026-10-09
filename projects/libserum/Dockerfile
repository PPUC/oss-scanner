# libserum: decoder for Serum colorization files for pinball ROMs.
# Mirrors the linux jobs in .github/workflows/libserum.yml, which run on ubuntu-24.04.
FROM ubuntu:24.04
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates build-essential cmake pkg-config git curl unzip python3 \
    && rm -rf /var/lib/apt/lists/*

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# The platform scripts are per architecture; pick the one matching the build machine.
RUN case "$(uname -m)" in \
      x86_64) echo x64 ;; \
      aarch64|arm64) echo aarch64 ;; \
      *) echo "unsupported architecture: $(uname -m)" >&2; exit 1 ;; \
    esac > /etc/ppuc-arch

# Fetch the one external dependency, the header-only FrameUtil.h from PPUC/libframeutil. cereal, lz4 and
# miniz are vendored in third-party/include. This is the only step that needs the network.
RUN ./platforms/linux/"$(cat /etc/ppuc-arch)"/external.sh

# The library (shared and static), the serum_test command-line loader and the unit tests, with debug info
# and frame pointers.
RUN cmake -S /src -B /src/build \
      -DPLATFORM=linux -DARCH="$(cat /etc/ppuc-arch)" \
      -DCMAKE_BUILD_TYPE=RelWithDebInfo \
      -DCMAKE_C_FLAGS="-g -fno-omit-frame-pointer" \
      -DCMAKE_CXX_FLAGS="-g -fno-omit-frame-pointer" \
    && cmake --build /src/build -j"$(nproc)"

# The same again under AddressSanitizer and UBSan (the project's own ENABLE_SANITIZERS option, which needs a
# Debug build and PLATFORM=linux to take effect; see tests/README.md).
RUN cmake -S /src -B /src/build-san \
      -DPLATFORM=linux -DARCH="$(cat /etc/ppuc-arch)" \
      -DCMAKE_BUILD_TYPE=Debug \
      -DENABLE_SANITIZERS=ON \
    && cmake --build /src/build-san -j"$(nproc)"

# Run the tests so a broken image is visible in the log, without letting a failing test abort the build.
# Leak detection needs ptrace, which `docker build` does not grant.
RUN (ctest --test-dir /src/build --output-on-failure --timeout 300 \
      || echo "WARNING: ctest reported failures") \
    && (ASAN_OPTIONS=detect_leaks=0 ctest --test-dir /src/build-san --output-on-failure --timeout 300 \
      || echo "WARNING: ctest reported failures under sanitizers")
