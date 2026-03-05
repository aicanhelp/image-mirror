# image-mirror
主要是解决获取某些镜像比较困难的问题，比如gcr.io上的镜像。

## 解决思路
利用github的workflow和skopeo工具, 把镜像copy到指定的registry，比如自己的gcrh.io或者阿里云的自己的镜像仓库。

## 使用方法
actions --> All workflows, 选择 image-mirror， 点击右边的 run-workflow, 输入参数：
- source_image（源镜像名称）
- source_tag （源镜像标签）
- target_image （目标镜像名称（默认同源））

当前构建的.github/workflows/image-mirror.yml的定义如下，主要用skopeo copy 镜像到当前项目的repository。可构建其他的workflow，把镜像copy到阿里云自己的镜像仓库。

```
name: image-mirror

on:
  workflow_dispatch:
    inputs:
      source_image:
        description: '源镜像名称（如 nginx, redis）'
        required: true
        default: 'nginx'
      source_tag:
        description: '源镜像标签'
        required: false
        default: 'latest'
      target_image:
        description: '目标镜像名称（默认同源）'
        required: false
        default: ''

jobs:
  copy-image:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Install Skopeo
        run: |
          sudo apt-get update
          sudo apt-get install -y skopeo

      - name: Extract target image name
        id: extract-name
        run: |
          if [[ -z "${{ github.event.inputs.target_image }}" ]]; then
            echo "target_image=${{ github.event.inputs.source_image }}" >> $GITHUB_OUTPUT
          else
            echo "target_image=${{ github.event.inputs.target_image }}" >> $GITHUB_OUTPUT
          fi

      - name: Copy multi-arch image with Skopeo
        run: |
          if echo "${{ github.event.inputs.source_image }}" | grep '.*\..*/.*' >/dev/null; then
             skopeo copy --all \
              docker://${{ github.event.inputs.source_image }}:${{ github.event.inputs.source_tag }} \
              docker://ghcr.io/${{ github.repository }}/${{ steps.extract-name.outputs.target_image }}:${{ github.event.inputs.source_tag }}
          else
             skopeo copy --all \
               docker://docker.io/${{ github.event.inputs.source_image }}:${{ github.event.inputs.source_tag }} \
               docker://ghcr.io/${{ github.repository }}/${{ steps.extract-name.outputs.target_image }}:${{ github.event.inputs.source_tag }}
          fi

      - name: Verify copied manifest
        run: |
          echo "✅ Verifying multi-arch manifest..."
          curl -H "Authorization: Bearer ${{ secrets.GITHUB_TOKEN }}" \
            https://ghcr.io/v2/${{ github.repository }}/${{ steps.extract-name.outputs.target_image }}/manifests/${{ github.event.inputs.source_tag }} | jq '.mediaType'

```
