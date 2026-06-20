# Multi-stage build: compile ebg from C++ source, then ship only the binary
ARG EBG_REF=0068141dd83f7eef3c056194ef3b77e1f2f6c017

FROM ubuntu:jammy AS builder

ARG EBG_REF
RUN apt-get update && apt-get install -y --no-install-recommends \
        git ca-certificates build-essential \
    && rm -rf /var/lib/apt/lists/*

RUN git clone https://github.com/pblischak/polyploid-genotyping.git /tmp/src && \
    cd /tmp/src && \
    git checkout "${EBG_REF}" && \
    cd ebg && \
    make && \
    ./ebg -h 2>&1 | head -5 || true

# Runtime image
FROM ubuntu:jammy

LABEL maintainer="gptiley@ncsu.edu"
LABEL description="Docker image for EBG (empirical Bayesian genotyping) polyploid genotyper"
LABEL version="1.0.0"
LABEL repository="https://github.com/pblischak/polyploid-genotyping"

RUN groupadd -r appgroup && useradd -r -g appgroup appuser

WORKDIR /app

COPY --from=builder /tmp/src/ebg/ebg /app/ebg
RUN chmod +x /app/ebg

# Test script: generates synthetic input matching the Makefile's test
# dimensions (n=100 individuals, l=10 loci, ploidy=4) and runs `ebg hwe`.
RUN cat > /app/run-tests.sh << 'EOF'
#!/bin/bash
set -e

cd /app

echo "=== Verifying ebg binary ==="
ls -lh /app/ebg
/app/ebg -h 2>&1 | head -10 || true

echo ""
echo "=== Generating synthetic test data (n=100, l=10, p=4) ==="
# tot.txt and alt.txt: 100 rows (individuals) x 10 cols (loci), whitespace-separated.
# err.txt: one error rate per line per locus.
# Use tot=20 with alt varied per cell so allele freqs stay in (0,1) and the EM
# has real signal (uniform/boundary data produce -inf log-likelihoods).
awk 'BEGIN{for(i=1;i<=100;i++){for(j=1;j<=10;j++)printf "20%s",(j<10?" ":"\n")}}' > tot.txt
awk 'BEGIN{for(i=1;i<=100;i++){for(j=1;j<=10;j++){a=((i*3+j*7)%13)+3;printf "%d%s",a,(j<10?" ":"\n")}}}' > alt.txt
awk 'BEGIN{for(j=1;j<=10;j++)print "0.01"}' > err.txt

echo "Input files:"
ls -lh tot.txt alt.txt err.txt
echo "tot.txt: $(wc -l < tot.txt) rows, $(awk 'NR==1{print NF}' tot.txt) cols"
echo "alt.txt: $(wc -l < alt.txt) rows, $(awk 'NR==1{print NF}' alt.txt) cols"
echo "err.txt: $(wc -l < err.txt) lines"

echo ""
echo "=== Running ebg hwe model ==="
/app/ebg hwe -n 100 -l 10 -p 4 -t tot.txt -a alt.txt -e err.txt --prefix test_out

if [ $? -eq 0 ]; then
    echo "✅ ebg hwe run completed without error"
else
    echo "❌ ebg hwe run FAILED"
    exit 1
fi

echo ""
echo "=== Verifying output files ==="
for f in test_out-genos.txt test_out-freqs.txt test_out-PL.txt; do
    if [ -s "/app/$f" ]; then
        echo "✅ $f exists and is non-empty ($(wc -l < /app/$f) lines)"
    else
        echo "❌ $f missing or empty"
        exit 1
    fi
done

echo ""
echo "=== All tests completed successfully ==="
ls -lh /app/test_out-*.txt
EOF

RUN chmod +x /app/run-tests.sh && chown -R appuser:appgroup /app

USER appuser

CMD ["/app/run-tests.sh"]
