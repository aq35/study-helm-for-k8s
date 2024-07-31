Docker Desktop上でKubernetesとHelmを使用して、Laravel 11 + Nginx + MySQL 8.0の環境を構築する手順を提供します。コンテナイメージは1から作成します。

まず、必要なディレクトリ構造を作成します：

```bash
mkdir -p laravel-k8s/{charts,docker}
cd laravel-k8s
```

1. Dockerfileの作成（Laravel 11 + PHP 8.2）

```bash
cat << EOF > docker/Dockerfile
FROM php:8.2-fpm

# Install dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    libpng-dev \
    libjpeg62-turbo-dev \
    libfreetype6-dev \
    locales \
    zip \
    jpegoptim optipng pngquant gifsicle \
    vim \
    unzip \
    git \
    curl \
    libzip-dev

# Clear cache
RUN apt-get clean && rm -rf /var/lib/apt/lists/*

# Install extensions
RUN docker-php-ext-install pdo_mysql zip exif pcntl
RUN docker-php-ext-configure gd --with-freetype --with-jpeg
RUN docker-php-ext-install gd

# Install composer
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer

# Set working directory
WORKDIR /var/www

# Install Laravel
RUN composer create-project --prefer-dist laravel/laravel:^11.0 .

# Copy existing application directory contents
COPY . /var/www

# Copy existing application directory permissions
COPY --chown=www-data:www-data . /var/www

# Change current user to www
USER www-data

# Expose port 9000 and start php-fpm server
EXPOSE 9000
CMD ["php-fpm"]
EOF
```

2. Nginxの設定ファイル作成

```bash
cat << EOF > docker/nginx.conf
server {
    listen 80;
    index index.php index.html;
    error_log  /var/log/nginx/error.log;
    access_log /var/log/nginx/access.log;
    root /var/www/public;
    location ~ \.php$ {
        try_files \$uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass php:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME \$document_root\$fastcgi_script_name;
        fastcgi_param PATH_INFO \$fastcgi_path_info;
    }
    location / {
        try_files \$uri \$uri/ /index.php?\$query_string;
        gzip_static on;
    }
}
EOF
```

3. Helmチャートの作成

```bash
helm create charts/laravel-app
```

4. Helmチャートの値ファイル（values.yaml）の更新

```bash
cat << EOF > charts/laravel-app/values.yaml
replicaCount: 1

image:
  repository: laravel-app
  pullPolicy: IfNotPresent
  tag: "latest"

imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

serviceAccount:
  create: true
  annotations: {}
  name: ""

podAnnotations: {}

podSecurityContext: {}

securityContext: {}

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false

resources: {}

autoscaling:
  enabled: false

nodeSelector: {}

tolerations: []

affinity: {}

nginx:
  image:
    repository: nginx
    tag: alpine
  config:
    nginx.conf: |
      server {
          listen 80;
          index index.php index.html;
          error_log  /var/log/nginx/error.log;
          access_log /var/log/nginx/access.log;
          root /var/www/public;
          location ~ \.php$ {
              try_files $uri =404;
              fastcgi_split_path_info ^(.+\.php)(/.+)$;
              fastcgi_pass 127.0.0.1:9000;
              fastcgi_index index.php;
              include fastcgi_params;
              fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
              fastcgi_param PATH_INFO $fastcgi_path_info;
          }
          location / {
              try_files $uri $uri/ /index.php?$query_string;
              gzip_static on;
          }
      }

mysql:
  enabled: true
  auth:
    database: laravel
    username: laravel
    password: secret
    rootPassword: secret
EOF
```

5. Helmチャートのテンプレート（deployment.yaml）の更新

```bash
cat << EOF > charts/laravel-app/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "laravel-app.fullname" . }}
  labels:
    {{- include "laravel-app.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "laravel-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      {{- with .Values.podAnnotations }}
      annotations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      labels:
        {{- include "laravel-app.selectorLabels" . | nindent 8 }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "laravel-app.serviceAccountName" . }}
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
      containers:
        - name: {{ .Chart.Name }}
          securityContext:
            {{- toYaml .Values.securityContext | nindent 12 }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: php-fpm
              containerPort: 9000
              protocol: TCP
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
        - name: nginx
          image: "{{ .Values.nginx.image.repository }}:{{ .Values.nginx.image.tag }}"
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
          volumeMounts:
            - name: nginx-config
              mountPath: /etc/nginx/conf.d/default.conf
              subPath: nginx.conf
      volumes:
        - name: nginx-config
          configMap:
            name: {{ include "laravel-app.fullname" . }}-nginx-config
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
EOF
```

6. Nginxの設定用ConfigMapテンプレートの作成

```bash
cat << EOF > charts/laravel-app/templates/nginx-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "laravel-app.fullname" . }}-nginx-config
  labels:
    {{- include "laravel-app.labels" . | nindent 4 }}
data:
  nginx.conf: |
    {{- .Values.nginx.config.nginx.conf | nindent 4 }}
EOF
```

7. アプリケーションのビルドとデプロイ

```bash
# Dockerイメージのビルド
docker build -t laravel-app:latest -f docker/Dockerfile .

# Kubernetesコンテキストの設定（Docker Desktop）
kubectl config use-context docker-desktop

# Helmチャートのインストール
helm install laravel-app ./charts/laravel-app
```

これで、Laravel 11 + Nginx + MySQL 8.0の環境がKubernetes上に構築されます。アプリケーションにアクセスするには、以下のコマンドを実行してポートフォワーディングを設定します：

```bash
kubectl port-forward svc/laravel-app 8080:80
```

その後、ブラウザで `http://localhost:8080` にアクセスすると、Laravelのウェルカムページが表示されるはずです。

注意：この設定は開発環境用です。本番環境では、セキュリティ設定やリソース割り当てなどを適切に調整する必要があります。