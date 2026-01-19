# Use an official OpenJDK runtime as the base image
FROM eclipse-temurin:11-jre

# Set metadata labels
LABEL maintainer="gptiley@ncsu.edu"
LABEL description="Docker image for H-PoPG haplotyping application"
LABEL version="v0.2.0"
LABEL repository="https://github.com/MinzhuXie/H-PoPG"

# Install git and wget for downloading the repository and health checks
RUN apt-get add --no-cache git wget

# Create a non-root user to run the application (security best practice)
RUN groupadd -S appgroup && useradd -S appuser -G appgroup

# Set the working directory inside the container
WORKDIR /app

# Clone the H-PoPG repository
RUN git clone https://github.com/MinzhuXie/H-PoPG.git /tmp/h-popg && \
    cp /tmp/h-popg/H-PoPGv0.2.0.jar /app/H-PoPGv0.2.0.jar && \
    cp /tmp/h-popg/frags.txt /app/frags.txt && \
    cp /tmp/h-popg/example.vcf /app/example.vcf && \
    cp -r /tmp/h-popg/bamvcfexample /app/bamvcfexample && \
    rm -rf /tmp/h-popg

# Change ownership of the application files to the non-root user
RUN chown -R appuser:appgroup /app

# Switch to the non-root user
USER appuser

# Set JVM options (optional but recommended)
ENV JAVA_OPTS="-Xms256m -Xmx512m"

# Create a test script to run the application with example data
RUN echo '#!/bin/bash' > /app/run-tests.sh && \
    echo 'echo "=== Running H-PoP Test (without VCF) ==="' >> /app/run-tests.sh && \
    echo 'java $JAVA_OPTS -jar /app/H-PoPGv0.2.0.jar -p 3 -f /app/frags.txt -o /app/test_output_hpop.txt' >> /app/run-tests.sh && \
    echo 'if [ $? -eq 0 ]; then echo "H-PoP test PASSED"; else echo "H-PoP test FAILED"; exit 1; fi' >> /app/run-tests.sh && \
    echo 'echo ""' >> /app/run-tests.sh && \
    echo 'echo "=== Running H-PoPG Test (with VCF) ==="' >> /app/run-tests.sh && \
    echo 'java $JAVA_OPTS -jar /app/H-PoPGv0.2.0.jar -p 3 -f /app/frags.txt -vcf /app/example.vcf -o /app/test_output_hpopg.txt' >> /app/run-tests.sh && \
    echo 'if [ $? -eq 0 ]; then echo "H-PoPG test PASSED"; else echo "H-PoPG test FAILED"; exit 1; fi' >> /app/run-tests.sh && \
    echo 'echo ""' >> /app/run-tests.sh && \
    echo 'echo "=== All tests completed successfully ==="' >> /app/run-tests.sh && \
    echo 'echo "Output files created:"' >> /app/run-tests.sh && \
    echo 'ls -lh /app/test_output_*.txt' >> /app/run-tests.sh && \
    chmod +x /app/run-tests.sh

# Default command runs the tests
CMD ["/app/run-tests.sh"]

# Alternative:  To run the application interactively, override the CMD: 
# docker run -it <image-name> java -jar /app/H-PoPGv0.2.0.jar -h
