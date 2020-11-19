_First section explains the WPE deployment workflow, while the second section is in regards to the WPE Sage 10 Blade template cache issue._

# Github Actions Workflow to Deploy Sage 10 on WP Engine

#### Inspired By:

  * [Linchpin's WP Engine Deploy w/ Github Actions](https://github.com/linchpin/action-wpengine-deploy)
  * [Sidas Snieška's CircleCI + WPEngine + SAGE Pipeline](https://gist.github.com/Prophe1/5c1f7ce79a27a5e0eb9d43512a573111)
  * [Ryan Hoover's CircleCI 2.1 Continuous Integration with WP Engine](https://gist.github.com/ryanshoover/ce4bb081b95168840b8c51b2a033c500)

## [.yml](https://github.com/gardinermichael/ga-wpe-sage-10-workflow/blob/main/.github/workflows/deploy.yml):

```yml

# Manually triggered workflow

name: Deploy Workflow

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

      - name: Deploy - Git Push # unlink .gitignore; ln -s .gitignores/__production .gitignore (Place on line ##)
        run: |
          git checkout -b ${GITHUB_REF##*/}-${{ github.run_id }}-${{ github.run_number }}
          
          git rm -r -q --cached --ignore-unmatch --force .github .gitignores composer.* gulpfile.babel.js package.json webpack.config.js yarn.lock .babelrc .editorconfig .eslintignore .eslintrc.json .stylelintrc tests
          git add .
          git status
          ls -la
          git commit -m "Deployment commit"
          git push wpe  ${GITHUB_REF##*/}-${{ github.run_id }}-${{ github.run_number }}
          git push wpe :${GITHUB_REF##*/}-${{ github.run_id }}-${{ github.run_number }} 



```

## Variables You Need To Set in Settings > Secrets:

| Variable | Value |
| ------------- | ------------- |
| PROJECT_TYPE | Either `theme` or `plugin`. Corresponds to file path. Workflow will add an `s`. See line ##. |
| PROJECT_NAME | Name of the theme/plugin folder. Corresponds to file path. See line ##. |
| WPE_INSTALL_PRODUCTION | Name of your WPE Production Server. Corresponds to [this](https://wpengine.com/support/git/#Git_Push_Deploy#Add_Git_Remotes). See line ##. |
| WPE_INSTALL_Staging | Name of your WPE Staging Server. Corresponds to [this](https://wpengine.com/support/git/#Git_Push_Deploy#Add_Git_Remotes). See line ##. __NOTE:__ You'll need to make a separate workflow for staging/production deploys, mostly so production can be triggered manually and staging can be triggered by pushing to the branch (Not shown here yet). Have to change the variable referenced on line ##. | 
| WPE_SSH_KNOWN_HOSTS | The git.wpengine.com fingerprint. For adding to known hosts. __NOTE:__ Will need to be updated if and when WP Engine changes their fingerprint. Value provided below. See line ##. |
| WPE_SSH_KEY_PRIVATE | The private SSH key you've generated on your machine and added to WPE. [Read this for further instructions](https://wpengine.com/support/git/#Git_Push_Deploy#Generate_SSH_Key). Should start with `-----BEGIN RSA PRIVATE KEY-----` and end with `-----END RSA PRIVATE KEY-----`. See line ##.

## Explanation:

This workflow assumes your repo lives at the theme or plugin folder level. It has three steps: Caching, Building and Deploying. __NOTE:__ See .gitignore section below for when setting up your repo.

### Caching 

Signaled by `Cache - ` in front of the step name. This process finds the cache directory path for both yarn and composer, and then in the subsequent step, loads it if it exists. I do not believe the cache hit commands commented out are necessary, but I left them there for posterity.

### Building

Signaled by `Build - ` in front of the step name. This process installs and builds composer and yarn packages/dependencies. Most importantly, moves all relevant files into a `wp-content/themes/project-name` or `wp-content/plugins/project-name` directory path to compensate for WPE pushing at the site folder level.

### Deploying 

Signaled by `Deploy - ` in front of the step name. This process configures the relevant SSH keys, adds the remote to git, switches out the .gitignore by delinking and relinking, and finally pushes to WPE.

## Addendum:

### .gitignore

You will have two .gitignores that live in a folder at your repo's root called `.gitignores`. The `__default` file corresponds to pushing from your local machine to Github, while the `__production` file corresponds to pushing from Github to WPE. When setting up your repo, add the default .gitignore with the following command at the repo root level: `ln -s .gitignores/__default .gitignore`. __NOTE:__ I need to finish the last step still (delinking and relinking the production .gitignore, so the workflow as it stands will add a bunch of unnecessary things to the WPE. Will fix soon. Please see [this folder](https://github.com/gardinermichael/ga-wpe-sage-10-workflow/tree/main/.gitignores) for the files.

### WPE SSH Fingerprint

See above section for explanation.

| Variable | Value |
| :------------- | :-------------|
| WPE_SSH_KNOWN_HOSTS | `git.wpengine.com ssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAQEApRVAUwjz49VKfuENfyv52Dvh3qx9nWW/3Gb7R9pwABXUNQqkipt3aB7w2W6jOaEGFmzSr/4qhstUv0lvbeZu/1uRU/b6WrqULu+9bAdt9ll09QULfMxAIFWDwDS1F6GEZT+Yau/wLUI2VTZppxSVRIPe20/mxgXk8/Q9ha5tCaz+dQZ9lHWwk9rbDF+7LSVomLGM3e9dwr6mS4p37Qkje2cFJBqQcQ+RqEOTOD/xiFU0DH8TWO4R5yibQ0KEZVACkwhaAZSl81F7YZrrLEfsFS/llgpV3YZHQGvFi0x/ELAUJMFE9umdy9EwFF7/lTpV8zOGdiLW+v8svweWJJJ00w==` |


---

# Sage 10 Blade Template Cache Solution for WP Engine


### The Problem

In Sage 9, the Blade template cache could be configured for WPE with the following: changing `'compiled' =>` to `'compiled' => '/tmp/sage-cache',` in `./config/view.php`. Relevant links:

  * [Sage 9 on WPEngine](https://discourse.roots.io/t/sage-9-on-wpengine/9090)
  * [Sage 9 and WPEngine Cache Fix](https://laneparton.com/sage-9-and-wpengine/)
  * [Tips For Setting up Sage 9](https://hirecollin.com/2019/08/tips-for-setting-up-sage-9/)
  
This will not work out of the box for Sage 10 unfortunately. [@jamesfacts](https://github.com/jamesfacts) realized that because Sage 10 has moved to utilizing Acorn for caching, for one reason or another, Sage no longer looks at the /tmp/ folder than we as WPE users can see in the SSH shell. Instead, it's pointing at a different /tmp/ folder that lives on the server root and is used by WPE proper. Only WPE support can see this folder.

### The Solution

You will need to contact WPE support and ask them to do the following:

```
Manually create the directory for it: 

mkdir /tmp/sage-cache

and then allow WP to write to it: 

find /tmp/sage-cache/ -type d -exec chown -R www-data: {} \;
find /tmp/sage-cache/ -type d -exec chmod -R 0775 {} \; 
```

__NOTE:__ Last but not least, you may have to replicate the process on your local machine. Not sure if it's the messed up permissions on my computer or if it was updating to Big Sur that caused it, but I had to manually create the `sage-cache` folder at `/tmp/` and give it the appropriate permissions as well.
