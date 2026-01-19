# Use non-Alpine base image for multi-platform support
FROM eclipse-temurin:11-jre

# Set metadata labels
LABEL maintainer="gptiley@ncsu.edu"
LABEL description="Docker image for H-PoPG haplotyping application"
LABEL version="0.2.0"
LABEL repository="https://github.com/MinzhuXie/H-PoPG"

# Install git and wget for downloading the repository
RUN apt-get update && apt-get install -y git wget && rm -rf /var/lib/apt/lists/*

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
set -e

echo "=== Verifying files exist ==="
ls -lh /app/*.jar /app/*.txt /app/*.vcf || true

echo ""
echo "=== Running H-PoP Test (without VCF) ==="
java $JAVA_OPTS -jar /app/H-PoPGv0.2.0.jar -p 3 -f /app/frags.txt -o /app/test_output_hpop.txt

if [ $? -eq 0 ]; then 
    echo "✅ H-PoP test PASSED"
else 
    echo "❌ H-PoP test FAILED"
    exit 1
fi

echo ""
echo "=== Running H-PoPG Test (with VCF) ==="
java $JAVA_OPTS -jar /app/H-PoPGv0.2.0.jar -p 3 -f /app/frags.txt -vcf /app/example.vcf -o /app/test_output_hpopg.txt

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
