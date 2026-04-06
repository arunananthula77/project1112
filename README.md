name: Deploy to Staging

on:
  push:
    branches:
      - staging

jobs:
  deploy-backend:
    name: Deploy Django Backend
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run tests
        run: python manage.py test

      - name: Deploy to Staging EC2 via SSH
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.STAGING_EC2_HOST }}
          username: ubuntu
          key: ${{ secrets.STAGING_EC2_SSH_KEY }}
          script: |
            cd /home/ubuntu/symplichain
            git pull origin staging
            source venv/bin/activate
            pip install -r requirements.txt
            python manage.py migrate --noinput
            python manage.py collectstatic --noinput
            sudo systemctl restart gunicorn
            sudo systemctl restart nginx

  deploy-frontend:
    name: Deploy React Frontend to S3
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - uses: actions/setup-node@v3
        with:
          node-version: '18'

      - name: Install and Build
        run: |
          cd frontend
          npm install
          npm run build
        env:
          REACT_APP_API_URL: ${{ secrets.STAGING_API_URL }}

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-south-1

      - name: Sync to S3
        run: |
          aws s3 sync frontend/build/ s3://${{ secrets.STAGING_S3_BUCKET }} --delete

      - name: Invalidate CloudFront cache
        run: |
          aws cloudfront create-invalidation \
            --distribution-id ${{ secrets.STAGING_CF_DISTRIBUTION_ID }} \
            --paths "/*"
