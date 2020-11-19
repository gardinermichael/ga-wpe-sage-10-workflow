# Github Actions Workflow to Deploy Sage 10 on WP Engine

#### Inspired By:

  * [Linchpin's WP Engine Deploy w/ Github Actions](https://github.com/linchpin/action-wpengine-deploy)
  * [Sidas Snieška's CircleCI + WPEngine + SAGE Pipeline](https://gist.github.com/Prophe1/5c1f7ce79a27a5e0eb9d43512a573111)
  * [Ryan Hoover's CircleCI 2.1 Continuous Integration with WP Engine](https://gist.github.com/ryanshoover/ce4bb081b95168840b8c51b2a033c500)

#### .yml:

```yml

# Manually triggered workflow

name: Production Deploy

on:
  workflow_dispatch:

jobs:
  Build-and-Deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Code
        uses: actions/checkout@v2
        with:
          fetch-depth: '0'
          
          
      - name: Cache - Get Yarn Cache Directory Path
        id: yarn-cache-dir-path
        run: echo "::set-output name=dir::$(yarn cache dir)"

      - name: Cache - Load Yarn Cache
        uses: actions/cache@v2
        id: yarn-cache # use this to check for `cache-hit` (`steps.yarn-cache.outputs.cache-hit != 'true'`)
        with:
          path: ${{ steps.yarn-cache-dir-path.outputs.dir }}
          key: ${{ runner.os }}-yarn-${{ hashFiles('**/yarn.lock') }}
          restore-keys: |
            ${{ runner.os }}-yarn-
      
      - name: Cache - Get Composer Cache Directory Path
        id: get-composer-cache-dir
        run: echo "::set-output name=dir::$(composer config cache-files-dir)"
      
      - name: Cache - Load Composer Cache
        uses: actions/cache@v2
        id: composer-cache # use this to check for `cache-hit` (`steps.composer-cache.outputs.cache-hit != 'true'`)
        with:
          path: ${{ steps.get-composer-cache-dir.outputs.dir }}
          key: ${{ runner.os }}-composer-${{ hashFiles('**/composer.lock') }}
          restore-keys: |
            ${{ runner.os }}-composer-
      - name: Build - Install Yarn Dependecies
        run: |
          yarn --prefer-offline
          yarn build
          yarn build:production
          
      - name: Build - Install Composer Dependencies
        run: composer install --no-ansi --no-dev --no-interaction --optimize-autoloader --no-progress

      - name: Build - Move files Into Project Folder
        run: |
          mkdir -p wp-content/${PROJECT_TYPE}s/${PROJECT_NAME}
          ls | grep -v wp-content | xargs mv -t wp-content/${PROJECT_TYPE}s/${PROJECT_NAME}
        env:
          PROJECT_TYPE: ${{secrets.PROJECT_TYPE}}
          PROJECT_NAME: ${{secrets.PROJECT_NAME}}


      - name: Deploy - Create SSH key
        run: |
          mkdir -p ~/.ssh/
          echo "$SSH_PRIVATE_KEY" > ~/.ssh/id_rsa
          sudo chmod 600 ~/.ssh/id_rsa
          echo "$SSH_KNOWN_HOSTS" > ~/.ssh/known_hosts
        env:
          SSH_PRIVATE_KEY: ${{ secrets.WPE_SSH_KEY_PRIVATE }}
          SSH_KNOWN_HOSTS: ${{ secrets.WPE_SSH_KNOWN_HOSTS }}

      - name: Deploy - Git Config
        run: |
           git remote add wpe git@git.wpengine.com:production/${WPE_INSTALL_PRODUCTION}.git
           git config --global user.email "actions@github.com"
           git config --global user.name "Github Action Deployment"
           git fetch wpe
        env:
          WPE_INSTALL_PRODUCTION: ${{ secrets.WPE_INSTALL_PRODUCTION }}

      - name: Deploy - Git Push # unlink .gitignore; ln -s .gitignores/__production .gitignore
        run: |
          git checkout -b ${GITHUB_REF##*/}-${{ github.run_id }}-${{ github.run_number }}
          
          git rm -r -q --cached --ignore-unmatch --force .github .gitignores composer.* gulpfile.babel.js package.json webpack.config.js yarn.lock .babelrc .editorconfig .eslintignore .eslintrc.json .stylelintrc codeception* tests
          git add .
          git status
          ls -la
          git commit -m "Deployment commit"
          git push wpe  ${GITHUB_REF##*/}-${{ github.run_id }}-${{ github.run_number }}
          git push wpe :${GITHUB_REF##*/}-${{ github.run_id }}-${{ github.run_number }} 



```


# Sage 10 Cache Solution for WP Engine
