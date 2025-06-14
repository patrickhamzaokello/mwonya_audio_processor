# Audio Processor

A production-ready audio processing system for streaming platforms that provides professional loudness normalization, AAC encoding, and HLS (HTTP Live Streaming) generation with AWS S3 integration.

## Features

- **Professional Loudness Normalization**: EBU R128 standard (-14 LUFS) for consistent audio levels
- **Multi-Quality HLS Streaming**: Adaptive bitrate streaming with low (48k), medium (64k), and high (128k) quality
- **AAC Encoding**: High-quality audio compression optimized for streaming
- **AWS S3 Integration**: Scalable cloud storage for raw and processed audio files
- **Database Tracking**: Complete processing status and metadata management
- **Automatic Cleanup**: Temporary file management and error recovery

## Requirements

### System Dependencies
```bash
# Ubuntu/Debian
sudo apt-get update
sudo apt-get install ffmpeg

# macOS
brew install ffmpeg

# CentOS/RHEL
sudo yum install epel-release
sudo yum install ffmpeg
```

### Python Dependencies
```bash
pip install boto3 ffmpeg-python mysql-connector-python psycopg2-binary
```

## Installation

1. **Clone or download the audio processor script**

2. **Install dependencies:**
```bash
pip install -r requirements.txt
```

3. **Set up environment variables:**
```bash
export AWS_ACCESS_KEY_ID="your_access_key_id"
export AWS_SECRET_ACCESS_KEY="your_secret_access_key"
export AWS_REGION="us-east-1"
```

4. **Configure your database** (MySQL or PostgreSQL)

## Configuration

### Environment Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `AWS_ACCESS_KEY_ID` | AWS access key | `AKIAIOSFODNN7EXAMPLE` |
| `AWS_SECRET_ACCESS_KEY` | AWS secret key | `wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY` |
| `AWS_REGION` | AWS region | `us-east-1` |

### Database Configuration

```python
# MySQL Configuration
db_config = {
    'host': 'localhost',
    'user': 'your_username',
    'password': 'your_password',
    'database': 'your_database',
    'port': 3306
}

# PostgreSQL Configuration
db_config = {
    'host': 'localhost',
    'user': 'your_username',
    'password': 'your_password',
    'database': 'your_database',
    'port': 5432
}
```

### S3 Bucket Structure
```
your-bucket/
├── tracks/                          # Raw audio files
├── mwonya_audio_tracks/            # Processed files
│   └── track_id/
│       ├── track_id.m4a            # AAC version
│       ├── playlist.m3u8           # Master playlist
│       ├── metadata.json           # Track metadata
│       ├── low/
│       │   ├── playlist.m3u8       # Low quality playlist
│       │   └── segment_*.ts        # Audio segments
│       ├── med/
│       │   ├── playlist.m3u8       # Medium quality playlist
│       │   └── segment_*.ts        # Audio segments
│       └── high/
│           ├── playlist.m3u8       # High quality playlist
│           └── segment_*.ts        # Audio segments
└── failedprocessing_tracks/        # Failed processing files
```

## Usage

### Basic Usage

```python
from audio_processor import AudioProcessor

# Initialize processor
processor = AudioProcessor(
    s3_bucket='your-bucket-name',
    db_config=db_config,
    db_type='mysql'  # or 'postgresql'
)

# Process a single track
success = processor.process_audio('track_filename.mp3')

# Process all pending tracks
successful, failed = processor.process_pending_tracks()
```

### Advanced Configuration

```python
# Custom configuration
processor = AudioProcessor(
    s3_bucket='your-bucket-name',
    db_config=db_config,
    db_type='mysql',
    raw_prefix='raw_audio/',
    processed_prefix='processed_audio/',
    failed_prefix='failed_audio/',
    segment_duration=15  # 15-second HLS segments
)
```

### Database Schema

Create the required database table:

```sql
CREATE TABLE Uploads (
    upload_id VARCHAR(255) PRIMARY KEY,
    file_name VARCHAR(255) NOT NULL,
    upload_type VARCHAR(50) DEFAULT 'track',
    processing_status VARCHAR(20) DEFAULT '0',
    uploaded_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    error_message TEXT
);
```

## Audio Processing Pipeline

### 1. Loudness Normalization
- **Standard**: EBU R128 (-14 LUFS)
- **Dynamic Range**: 11 LU
- **True Peak Limit**: -1.5 dBFS
- **Purpose**: Consistent perceived loudness across all tracks

### 2. AAC Encoding
- **Bitrate**: 128 kbps
- **Sample Rate**: 44.1 kHz
- **Channels**: Stereo
- **Container**: M4A

### 3. HLS Generation
- **Segment Duration**: 10 seconds (configurable)
- **Quality Levels**:
  - **Low**: 48 kbps, 24 kHz (mobile/low bandwidth)
  - **Medium**: 64 kbps, 32 kHz (standard streaming)
  - **High**: 128 kbps, 44.1 kHz (high quality)

## API Reference

### AudioProcessor Class

#### Constructor
```python
AudioProcessor(s3_bucket, db_config, db_type='mysql', raw_prefix='tracks/', 
               processed_prefix='mwonya_audio_tracks/', 
               failed_prefix='failedprocessing_tracks/', segment_duration=10)
```

#### Methods

##### `process_audio(track_id)`
Process a single audio file.
- **Parameters**: `track_id` (str) - Filename of the track to process
- **Returns**: `bool` - True if successful, False otherwise

##### `process_pending_tracks()`
Process all pending tracks from the database.
- **Returns**: `tuple` - (successful_count, failed_count)

##### `get_pending_tracks()`
Get list of tracks awaiting processing.
- **Returns**: `list` - List of track dictionaries

## Monitoring and Logging

The system provides comprehensive logging for production monitoring:

```python
import logging

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
```

### Key Log Events
- Processing start/completion
- File upload/download progress
- Loudness normalization results
- HLS generation status
- Error conditions and recovery

## Error Handling

### Database Status Tracking
- `'0'` - Pending processing
- `'processing'` - Currently being processed
- `'completed'` - Successfully processed
- `'failed'` - Processing failed (with error message)

### Automatic Cleanup
- Temporary files removed after processing
- Failed uploads moved to designated S3 prefix
- Database status updated on all outcomes

## Performance Considerations

### Resource Requirements
- **CPU**: Intensive during loudness normalization and encoding
- **Memory**: ~100MB per concurrent processing job
- **Storage**: Temporary files require 2-3x original file size during processing
- **Network**: S3 upload/download bandwidth dependent

### Optimization Tips
1. **Concurrent Processing**: Use separate processes for multiple tracks
2. **Disk Space**: Monitor `/tmp` directory usage
3. **S3 Transfers**: Use multipart uploads for large files
4. **Database**: Implement connection pooling

## Troubleshooting

### Common Issues

#### FFmpeg Not Found
```bash
# Install FFmpeg
sudo apt-get install ffmpeg
```

#### AWS Credentials Error
```bash
# Set credentials
export AWS_ACCESS_KEY_ID="your_key"
export AWS_SECRET_ACCESS_KEY="your_secret"

# Or use IAM roles (recommended for production)
```

#### Database Connection Issues
- Verify database credentials and connectivity
- Check firewall settings
- Ensure database exists and table is created

#### S3 Permission Errors
Required S3 permissions:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:DeleteObject"
            ],
            "Resource": "arn:aws:s3:::your-bucket/*"
        }
    ]
}
```

## Security Best Practices

1. **Use IAM Roles**: Avoid hardcoded AWS credentials
2. **Encrypt S3 Buckets**: Enable server-side encryption
3. **Database Security**: Use connection encryption and proper authentication
4. **Input Validation**: Validate all track IDs and filenames
5. **Network Security**: Use VPC for database connections

## Production Deployment

### Docker Example
```dockerfile
FROM python:3.9-slim

RUN apt-get update && apt-get install -y ffmpeg

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY audio_processor.py .
COPY database_manager.py .

CMD ["python", "audio_processor.py"]
```

### Kubernetes Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: audio-processor
spec:
  replicas: 3
  selector:
    matchLabels:
      app: audio-processor
  template:
    metadata:
      labels:
        app: audio-processor
    spec:
      containers:
      - name: audio-processor
        image: your-registry/audio-processor:latest
        env:
        - name: AWS_ACCESS_KEY_ID
          valueFrom:
            secretKeyRef:
              name: aws-secrets
              key: access-key-id
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

## License

This project is licensed under the MIT License.

## Support

For issues and questions:
1. Check the troubleshooting section
2. Review logs for error details
3. Verify all dependencies are installed
4. Ensure proper AWS and database permissions

## Changelog

### v1.0.0
- Initial release with EBU R128 loudness normalization
- HLS streaming support with adaptive bitrates
- AWS S3 integration
- Database tracking and error handling
