public:: true

- ## 使用 firebase publish 的原因
	- 原本使用 https://github.com/logseq/publish-spa 發現無法顯示圖片
- ## 設定步驟
	- 1. Create new Github repo
	- 2. Logseq create new graph 指定本地位置到 git clone 的路徑
	- 3. Logseq setting
	  ```
	  :editor/preferred-pasting-file? true
	  :publishing/all-pages-public? true
	  ```
	- 4. Logseq enable auto commit (Optional)
	- 5. Firebase 設定，透過 CLI 建立 hosting
		- 5-1. `sudo npm install -g firebase-tools`
		- 5-2. `firebase login`
		- 5-3. `firebase init hosting`
		  ![image.png](../assets/image_1719212096536_0.png)
		  ![image.png](../assets/image_1719212150032_0.png)modify `.firebaserc` by command 
		  ```
		  // firebase target:apply hosting <target> <resources...>
		  firebase target:apply hosting note-e3304 note-e3304
		  ```
	- 6. edit `firebase.json`，設定 target, public, destination
	  ```
	  {
	    "hosting": {
	      "target": "note-e3304",
	      "public": "www",
	      "ignore": [
	        "firebase.json",
	        "**/.*",
	        "**/node_modules/**"
	      ],
	      "rewrites": [
	        {
	          "source": "**",
	          "destination": "www/index.html"
	        }
	      ]
	    }
	  }
	  ```
	- 7. 建立 `.github/workflows/publish.yml` replace firebaseServiceAccount and projectId from `.github/workflows/firebase-hosting-pull-request.yml`
	  ```
	  # This is a basic workflow to help you get started with Actions
	  name: Logseq Deploy
	  
	  # Controls when the workflow will run
	  on:
	    push:
	      branches: [main]
	  
	    # Allows you to run this workflow manually from the Actions tab
	    workflow_dispatch:
	    # A workflow run is made up of one or more jobs that can run sequentially or in parallel
	  
	  permissions:
	      contents: write
	  
	  jobs:
	    # This workflow contains a single job called "build"
	    build:
	      # The type of runner that the job will run on
	      runs-on: ubuntu-latest
	  
	      # Steps represent a sequence of tasks that will be executed as part of the job
	      steps:
	        # Checks-out your repository under $GITHUB_WORKSPACE, so your job can access it
	        - uses: actions/checkout@v2
	        
	        # Publish action by other contributor
	        - name: Logseq Publish
	          uses: pengx17/logseq-publish@main
	          with:
	            version: latest
	  
	        - name: add a nojekyll file
	          run: touch $GITHUB_WORKSPACE/www/.nojekyll
	  
	        - name: Deploy to GitHub pages
	          uses: JamesIves/github-pages-deploy-action@v4
	          with:
	            branch: gh-pages # The branch the action should deploy to.
	            folder: www # The folder the action should deploy.
	            clean: true
	  
	        - name: deploy to firebase hosting live channel
	          uses: FirebaseExtended/action-hosting-deploy@v0
	          with:
	            repoToken: "${{ secrets.GITHUB_TOKEN }}"
	            firebaseServiceAccount: "${{ secrets.FIREBASE_SERVICE_ACCOUNT_NOTE_E3304 }}"
	            projectId: note-e3304
	            channelId: live
	  ```
	- 8. remove `.github/workflows/firebase-hosting-pull-request.yml`
	- 9. push `.firebaserc`, `firebase.json` and `.github/workflows/publish.yml`
	- 10. 看 Github action 有沒有成功
	  ![image.png](../assets/image_1719213052775_0.png)