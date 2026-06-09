# Reproducible build of the updog R package.
# Three pins control what gets baked in: R version (image tag), the PPM
# snapshot date (locks every CRAN dep), and the updog version itself.
# Bump any one of them in isolation to upgrade.
ARG R_VERSION=4.4.2
ARG PPM_SNAPSHOT=2026-05-01
ARG UPDOG_VERSION=2.1.7

FROM rocker/r-ver:${R_VERSION}

ARG PPM_SNAPSHOT
ARG UPDOG_VERSION

LABEL maintainer="gptiley@ncsu.edu"
LABEL description="Docker image for updog (flexible genotyping for polyploids)"
LABEL version="2.1.7"
LABEL repository="https://github.com/dcgerard/updog"

# System libraries needed to compile updog's C++ code (RcppArmadillo -> BLAS/LAPACK)
# and to satisfy its R dependency chain.
RUN apt-get update && apt-get install -y --no-install-recommends \
        libopenblas-dev \
        liblapack-dev \
        libxml2-dev \
        libssl-dev \
        libcurl4-openssl-dev \
        libgit2-dev \
    && rm -rf /var/lib/apt/lists/*

# Point R at a date-pinned PPM snapshot so every install resolves to the same
# package versions on every rebuild. rocker/r-ver:4.4.2 is Ubuntu 24.04 (noble).
RUN echo "options(repos = c(CRAN = 'https://packagemanager.posit.co/cran/__linux__/noble/${PPM_SNAPSHOT}'))" \
        >> /usr/local/lib/R/etc/Rprofile.site

# Install updog at a pinned version. remotes::install_version pulls from the
# repos set above, so deps are also resolved against the snapshot.
RUN R -e "install.packages('remotes')" && \
    R -e "remotes::install_version('updog', version = '${UPDOG_VERSION}', \
            upgrade = 'never', \
            dependencies = c('Depends', 'Imports', 'LinkingTo'))" && \
    R -e "stopifnot(packageVersion('updog') == '${UPDOG_VERSION}')"

RUN groupadd -r appgroup && useradd -r -g appgroup appuser

WORKDIR /app

RUN cat > /app/smoke-test.R << 'EOF'
library(updog)
cat("updog version:", as.character(packageVersion("updog")), "\n")
cat("R version:", R.version.string, "\n")

set.seed(1)

# 1. flexdog single-SNP fit — exercises the compiled C++ EM kernels.
refvec  <- c(1:20); refvec[2] <- NA
sizevec <- c(40:21); sizevec[3] <- NA
fout <- flexdog(refvec = refvec, sizevec = sizevec,
                ploidy = 4, model = "norm", verbose = FALSE)
stopifnot(is.list(fout),
          !is.null(fout$postmat),
          nrow(fout$postmat) == length(refvec),
          ncol(fout$postmat) == 5)
cat(sprintf("flexdog OK — bias=%.4f od=%.4f seq=%.4f\n",
            fout$bias, fout$od, fout$seq))

# 2. multidog multi-SNP fit — exercises the higher-level wrapper and
#    the foreach/future parallel plumbing.
refmat  <- matrix(c(5, 10, 15, 2, 8, 12), nrow = 2)
sizemat <- matrix(c(20, 20, 20, 20, 20, 20), nrow = 2)
rownames(refmat)  <- c("snp1", "snp2")
rownames(sizemat) <- c("snp1", "snp2")
colnames(refmat)  <- c("ind1", "ind2", "ind3")
colnames(sizemat) <- c("ind1", "ind2", "ind3")
mout <- multidog(refmat = refmat, sizemat = sizemat,
                 ploidy = 4, model = "norm", nc = 1)
stopifnot(nrow(mout$snpdf) == 2,
          "geno" %in% colnames(mout$inddf))
cat("multidog OK — snps:", nrow(mout$snpdf),
    " individual calls:", nrow(mout$inddf), "\n")

cat("All updog tests passed.\n")
EOF

RUN cat > /app/run-tests.sh << 'EOF'
#!/bin/bash
set -e

echo "=== updog smoke test ==="
Rscript /app/smoke-test.R

if [ $? -eq 0 ]; then
    echo "✅ updog tests PASSED"
else
    echo "❌ updog tests FAILED"
    exit 1
fi
EOF

RUN chmod +x /app/run-tests.sh && chown -R appuser:appgroup /app

USER appuser

CMD ["/app/run-tests.sh"]
