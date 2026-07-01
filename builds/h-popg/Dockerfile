# Debian-based JRE. Chosen over distroless / alpine because Nextflow's
# in-container trace collection needs a normal GNU userland: ps (procps),
# GNU date with %3N truncation (coreutils), bash, and friends.
#
# Pinned to the -jammy variant (Ubuntu 22.04 LTS) rather than the rolling
# `:17-jre` tag on purpose: newer Ubuntu (25.10+) ships uutils-coreutils
# (a Rust rewrite) whose `date +%3N` does not truncate to 3 digits, which
# breaks nxf_trace_linux's millisecond timestamp math silently.
FROM eclipse-temurin:17-jre-jammy

# Set metadata labels
LABEL maintainer="gptiley@ncsu.edu"
LABEL description="Docker image for H-PoPG haplotyping application"
LABEL version="0.2.0"
LABEL repository="https://github.com/MinzhuXie/H-PoPG"

# git + wget: fetch the H-PoPG release from GitHub at build time.
# procps + coreutils + grep + gawk + sed + bash: required by Nextflow's
# nxf_trace_linux / nxf_write_trace helpers in .command.run. Under the
# pipeline's strict shell (bash -C -e -u -o pipefail), a missing tool
# aborts the task before H-PoPG runs. Listing them explicitly makes the
# dependency intentional even though most are already in this base.
RUN apt-get update && apt-get install -y --no-install-recommends \
        git \
        wget \
        procps \
        coreutils \
        grep \
        gawk \
        sed \
        bash \
    && rm -rf /var/lib/apt/lists/*

# Create a non-root user to run the application (security best practice)
RUN groupadd -r appgroup && useradd -r -g appgroup appuser

# Set the working directory inside the container
WORKDIR /app

# Clone the H-PoPG repository and copy files
RUN git clone https://github.com/MinzhuXie/H-PoPG.git /tmp/h-popg && \
    ls -la /tmp/h-popg/ && \
    cp /tmp/h-popg/H-PoPGv0.2.0.jar /app/H-PoPGv0.2.0.jar && \
    cp /tmp/h-popg/frags.txt /app/frags.txt && \
    cp /tmp/h-popg/example.vcf /app/example.vcf && \
    cp -r /tmp/h-popg/bamvcfexample /app/bamvcfexample && \
    ls -la /app/ && \
    rm -rf /tmp/h-popg

# Create a test script BEFORE changing ownership
RUN cat > /app/run-tests.sh << 'EOF'
#!/bin/bash
set -euo pipefail

echo "=== Verifying files exist ==="
ls -lh /app/*.jar /app/*.txt /app/*.vcf || true

echo ""
echo "=== Verifying Nextflow trace prerequisites ==="
# Same tools nxf_trace_linux / nxf_write_trace rely on. If any of these
# don't behave, the image will fail Nextflow's task metrics collection
# under `bash -C -e -u -o pipefail` before H-PoPG runs.
command -v ps    >/dev/null && echo "  ps: $(ps --version | head -1)"
command -v awk   >/dev/null && echo "  awk: $(awk --version 2>&1 | head -1)"
command -v grep  >/dev/null && echo "  grep: $(grep --version | head -1)"
command -v sed   >/dev/null && echo "  sed: $(sed --version | head -1)"
command -v bash  >/dev/null && echo "  bash: $(bash --version | head -1)"
# GNU date %3N is what Nextflow uses for millisecond timestamps.
# uutils-coreutils accepts %3N syntactically but doesn't truncate, so
# check both: %N expanded (not busybox) AND %3N is exactly 3 digits.
n3="$(date +%3N)"
case "$n3" in
    *N)      echo "❌ date does not expand %N (not GNU or uutils)"; exit 1 ;;
esac
if [ "${#n3}" -ne 3 ]; then
    echo "❌ date +%3N returned ${#n3} chars ('$n3'), expected 3 (uutils-coreutils in the base?)"
    exit 1
fi
echo "  date +%3N truncates to 3 digits: $n3"
echo "  date +%s%3N unix ms: $(date +%s%3N)"
# /proc must be mounted and readable for ps and nxf_trace_linux to see PIDs.
if [ -r /proc/self/stat ]; then
    echo "  /proc readable: yes"
else
    echo "❌ /proc not readable"; exit 1
fi

echo ""
echo "=== Running H-PoP Test (without VCF) ==="
java ${JAVA_OPTS:-} -jar /app/H-PoPGv0.2.0.jar -p 3 -f /app/frags.txt -o /app/test_output_hpop.txt

if [ $? -eq 0 ]; then
    echo "✅ H-PoP test PASSED"
else
    echo "❌ H-PoP test FAILED"
    exit 1
fi

echo ""
echo "=== Running H-PoPG Test (with VCF) ==="
java ${JAVA_OPTS:-} -jar /app/H-PoPGv0.2.0.jar -p 3 -f /app/frags.txt -vcf /app/example.vcf -o /app/test_output_hpopg.txt

if [ $? -eq 0 ]; then
    echo "✅ H-PoPG test PASSED"
else
    echo "❌ H-PoPG test FAILED"
    exit 1
fi

echo ""
echo "=== All tests completed successfully ==="
echo "Output files created:"
ls -lh /app/test_output_*.txt || echo "No output files found"
EOF

# Make script executable
RUN chmod +x /app/run-tests.sh

# Change ownership of the application files to the non-root user
RUN chown -R appuser:appgroup /app

# Switch to the non-root user
USER appuser

# Set JVM options (optional but recommended)
ENV JAVA_OPTS="-Xms256m -Xmx512m"

# Default command runs the tests
CMD ["/app/run-tests.sh"]
