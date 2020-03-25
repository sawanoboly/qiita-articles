<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

最近はOpen-source software (OSS) に頼りきりなので、更新があった時に差分をチェックしたい。

[Grit(RubyGems)](http://rubygems.org/gems/grit)で簡単に取れそうなのでやってみた。

## 対象のOSSProjectを選ぶ

`Grit`はリモートのリポジトリにを直接見ることが出来なさそうなので、一旦サンプルのOSSProjectをローカルにコピーした。

今回のチョイスは近所のよしみで**`concrete5`**です。


## タグを取る

`./concrete5` ディレクトリにクローンしたものとします。
`#tag`ですべてのタグが`Grit::Tag`オブジェクトに突っ込まれました。

```ruby:rubycode
require 'grit'
                                                                                                                                                                               
repo = Grit::Repo.new('concrete5')
repo.tags.map {|t| puts t.name}

=>
5.6.1.2
5.6.1.1
5.6.1
5.6.0.2
5.6.0.1
5.6.0
5.5.2.1
5.5.2
5.5.1
5.5.0
5.4.2.2
5.4.2.1
5.4.2
```

`5.6.1.2`と`5.6.1.1`の差分が気になる体で行きましょう。


## コミット一覧

`#commits`では`Grit::Commit`オブジェクトがとれました、新しい順ですね。


```ruby:rubycode
repo.commits('5.6.1.2')

=> [#<Grit::Commit "b5b03cfdeed7cb512ea75403574599553472102a">,
 #<Grit::Commit "5b0e2774e8991d236657908208e7be55962697d3">,
 #<Grit::Commit "7dbc852e3823164374a54fd4a81ce3697ceea6b6">,
 #<Grit::Commit "9ac5b5413782d191b8c6b120a56761c0c89d93bb">,
 #<Grit::Commit "a7c323d0c57dec85a26ea2bc7b390c73e0c1adbe">,
 #<Grit::Commit "8267a2083d3a5b54aedda792a5666ab2a34f05f9">,
 #<Grit::Commit "c1f7e53cf2d151515d81e11770103a9c3f2b1e41">,
 #<Grit::Commit "0900e17de378e76adf1a4be5ce7db6ec46681aa7">,
 #<Grit::Commit "850f05d7acf93cddc97a24877783aaeb2a3a3190">,
 #<Grit::Commit "3f1aa9dadf12966c52a083c69e128c7fe97c0b79">]
```

デフォルトは最新10件です。


## diffを取る
比較する`Grit::Commit`のオブジェクトを`#diff`にかけると、ファイルごとの差分である`Grit::Diff`オブジェクトが取得できました。

```ruby:rubycode
new_commit = repo.commits('5.6.1.2')
old_commit = repo.commits('5.6.1.1')

repo.diff(new_commit[0], old_commit[0])

=> [#<Grit::Diff:0x00000002b67248
  @a_blob=#<Grit::Blob "fdf691e">,
  @a_mode=nil,
  @a_path="web/concrete/blocks/date_nav/auto.js",
  @b_blob=#<Grit::Blob "704f60b">,
  @b_mode="100644",
  @b_path="web/concrete/blocks/date_nav/auto.js",
  @deleted_file=false,
  @diff=
   "--- a/web/concrete/blocks/date_nav/auto.js\n+++ b/web/concrete/blocks/date_nav/auto.js\n@@ -4,18 +4,18 @@ var dateNav ={\n \t\tthis.blockForm=document.forms['ccm-block-for
  @new_file=false,
  @renamed_file=false,
  @repo=#<Grit::Repo "/home/higan/Dev/git-check/concrete5/.git">,
  @similarity_index=0>,
```

`@a_path`,`@b_path`,`@new_file`,`@deleted_file`,`@ renamed_file` あたりが気になりますね。

ざっと一覧で見れるように表示します。

```ruby:rubycode
diffs = repo.diff(new_commit[0], old_commit[0])
diffs.map do |d|
  puts 'a:' + d.a_path
  puts 'b:' + d.b_path
  puts '  New? : ' + d.new_file.to_s
  puts '  Deleted? : ' + d.deleted_file.to_s
  puts '  Renamed? : ' + d.renamed_file.to_s
end

=>
a:web/concrete/blocks/date_nav/auto.js
b:web/concrete/blocks/date_nav/auto.js
  New? : false
  Deleted? : false
  Renamed? : false
a:web/concrete/config/base.php
b:web/concrete/config/base.php
  New? : false
  Deleted? : false
  Renamed? : false
a:web/concrete/config/version.php
b:web/concrete/config/version.php
  New? : false
  Deleted? : false
  Renamed? : false
a:web/concrete/core/controllers/single_pages/dashboard/pages/types.php
b:web/concrete/core/controllers/single_pages/dashboard/pages/types.php
  New? : false
  Deleted? : false
  Renamed? : false
a:web/concrete/core/controllers/single_pages/dashboard/pages/types/add.php
b:web/concrete/core/controllers/single_pages/dashboard/pages/types/add.php
  New? : false
  Deleted? : false
  Renamed? : false
a:web/concrete/core/helpers/text.php
b:web/concrete/core/helpers/text.php
  New? : false
  Deleted? : false
  Renamed? : false
a:web/concrete/core/libraries/page_cache/types/file.php
b:web/concrete/core/libraries/page_cache/types/file.php
  New? : false
  Deleted? : false
  Renamed? : false
a:web/concrete/core/models/page_theme.php
b:web/concrete/core/models/page_theme.php
  New? : false
  Deleted? : false
  Renamed? : false
a:web/concrete/js/ccm.app.js
b:web/concrete/js/ccm.app.js
  New? : false
  Deleted? : false
  Renamed? : false
a:web/concrete/js/ccm_app/legacy_dialog.js
b:web/concrete/js/ccm_app/legacy_dialog.js
  New? : false
  Deleted? : false
  Renamed? : false
a:web/concrete/tools/pages/url_slug.php
b:web/concrete/tools/pages/url_slug.php
  New? : false
  Deleted? : false
  Renamed? : false
```

ファイル構成には変更が無いみたい。

## Diff付きで表示する

差分情報は`Grit::Diff`オブジェクトの`@diff`に入っています。
Syntaxhighlightの関係で全部`puts`にしたり、出力を一部加工しています。

追記：このサンプル、NewとOldが逆です。

```ruby:rubycode
diffs.map do |d|
  puts '## ' + d.a_path
  puts '### ' + d.b_path
  puts '**New File!**' if d.new_file
  puts '**Deleted!**' if d.deleted_file
  puts '**Renamed!!**' if d.renamed_file
  puts "### Diff"
  puts '```'
  puts d.diff unless d.deleted_file
  puts '```'
  puts '----'
end
----

=>
## web/concrete/blocks/date_nav/auto.js
### web/concrete/blocks/date_nav/auto.js

### Diff
\```
--- a/web/concrete/blocks/date_nav/auto.js
+++ b/web/concrete/blocks/date_nav/auto.js
@@ -4,18 +4,18 @@ var dateNav ={
 		this.blockForm=document.forms['ccm-block-form'];
 		
 		$('#cParentIDLocation').change(function() {
-			if($('#cParentIDLocation').val() === 'CUSTOM') {
+			if($('#cParentIDLocation').val() == 'CUSTOM') {
 			
 			}
 			
 		});
-		if (value === "custom") {
-			$("#ccm-autonav-page-selector").css('display','block');
-		} else {
-			$("#ccm-autonav-page-selector").hide();
+		if (value == "custom") {
+				$("#ccm-autonav-page-selector").css('display','block');
+			} else {
+				$("#ccm-autonav-page-selector").hide();
+			}
 		}

--snip --
 		
```


## MarkDownにする

引数を取れるようにしてMarkDown形式にして表示するサンプルスクリプトを用意しました。
puts の嵐ですいません。

追記：qiita用にdiffのシンタックスハイライト入れた。
追記：diffがnilだったらコケたので修正
追記：NewとOldが逆だったので直した。

```ruby:sample.rb
require 'grit'

repo = Grit::Repo.new(ARGV[0])

old_commit = repo.commits(ARGV[1])
new_commit = repo.commits(ARGV[2])

diffs = repo.diff(old_commit[0], new_commit[0])

puts "# The diff of #{ARGV[0]} between #{ARGV[1]} to #{ARGV[2]}"
puts "\n----\n"

diffs.map do |d|
  puts '## ' + d.a_path + "\n"
  puts '### ' + d.b_path + "\n"
  puts '**New File!**  ' if d.new_file
  puts '**Deleted!**  ' if d.deleted_file
  puts '**Renamed!!**  ' if d.renamed_file
  if d.diff
    puts "\n### Diff\n"
    puts '```diff:'
    puts d.diff[0..2000]
    puts '```'
    puts "\n** Too large, sniped!! **\n" if d.diff.size > 2000
  end
  puts "\n----\n"
end

```


これをTumblrにメールで投稿しようとしたがうまく表示されなかったのでここに貼ってしまおう。

実行はこの様に、ディレクトリとタグなりブランチ(`origin/branch_name`で渡す)なり、sha1をパスすればOK.

`$ ruby ./sample.rb concrete5 5.6.1.1 5.6.1.2`

herokuあたりで適当に自動実行させておきたいですね。

----
以下出力

# The diff of concrete5 between 5.6.1.1 to 5.6.1.2

----
## web/concrete/blocks/date_nav/auto.js
### web/concrete/blocks/date_nav/auto.js

### Diff
```diff:
--- a/web/concrete/blocks/date_nav/auto.js
+++ b/web/concrete/blocks/date_nav/auto.js
@@ -4,18 +4,18 @@ var dateNav ={
 		this.blockForm=document.forms['ccm-block-form'];
 		
 		$('#cParentIDLocation').change(function() {
-			if($('#cParentIDLocation').val() == 'CUSTOM') {
+			if($('#cParentIDLocation').val() === 'CUSTOM') {
 			
 			}
 			
 		});
-		if (value == "custom") {
-				$("#ccm-autonav-page-selector").css('display','block');
-			} else {
-				$("#ccm-autonav-page-selector").hide();
-			}
+		if (value === "custom") {
+			$("#ccm-autonav-page-selector").css('display','block');
+		} else {
+			$("#ccm-autonav-page-selector").hide();
 		}
 		
+		
 		/*
 		this.cParentIDRadios=this.blockForm.cParentID;
 		for(var i=0;i<this.cParentIDRadios.length;i++){
@@ -41,7 +41,7 @@ var dateNav ={
 	}, 
 	showDescriptionOpts:function(){ 
 		for(var i=0;i<this.showDescriptionsRadios.length;i++){
-			if( this.showDescriptionsRadios[i].checked && this.showDescriptionsRadios[i].value=='1' ){
+			if( this.showDescriptionsRadios[i].checked && this.showDescriptionsRadios[i].value==='1' ){
 				$('div#ccm-pagelist-summariesOptsWrap').css('display','block'); 
 				return; 
 			}				
@@ -72,7 +72,7 @@ var dateNav ={
 	}, 	
 	locationOtherShown:function(){
 		for(var i=0;i<this.cParentIDRadios.length;i++){
-			if( this.cParentIDRadios[i].checked && this.cParentIDRadios[i].value=='OTHER' ){
+			if( this.cParentIDRadios[i].checked && this.cParentIDRadios[i].value==='OTHER' ){
 				$('div.ccm-page-list-page-other').css('display','block');
 				return; 
 			}				
@@ -83,7 +83,7 @@ var dateNav ={
 		
 		var failed=0; 
 		
-		if( failed==1 ){
+		if( failed===1 ){
 			ccm_isBlockError=1;
 			return false;
 		}
@@ -93,4 +93,4 @@ var dateNav ={
 }
 $(function(){ dateNav.init(); });
 
-ccmValidateBlockForm = function() { return dateNav.validate(); }
\ No newline at end of file
+ccmValidateBlockForm = function() { return dateNav.validate(); }
```

----
## web/concrete/config/base.php
### web/concrete/config/base.php

### Diff
```diff:
--- a/web/concrete/config/base.php
+++ b/web/concrete/config/base.php
@@ -93,6 +93,11 @@ if (!defined("PAGE_PATH_SEPARATOR")) {
 	define('PAGE_PATH_SEPARATOR', '-');
 }
 
+if (!defined('PAGE_PATH_SEGMENT_MAX_LENGTH')) {
+	define('PAGE_PATH_SEGMENT_MAX_LENGTH', '128');
+}
+
+
 if (!defined('ENABLE_ASSET_COMPRESSION')) {
 	define('ENABLE_ASSET_COMPRESSION', false);
 }
```

----
## web/concrete/config/version.php
### web/concrete/config/version.php

### Diff
```diff:
--- a/web/concrete/config/version.php
+++ b/web/concrete/config/version.php
@@ -1,3 +1,3 @@
 <?
 defined('C5_EXECUTE') or die("Access Denied.");
-$APP_VERSION = '5.6.1.1';
\ No newline at end of file
+$APP_VERSION = '5.6.1.2';
\ No newline at end of file
```

----
## web/concrete/core/controllers/single_pages/dashboard/pages/types.php
### web/concrete/core/controllers/single_pages/dashboard/pages/types.php

### Diff
```diff:
--- a/web/concrete/core/controllers/single_pages/dashboard/pages/types.php
+++ b/web/concrete/core/controllers/single_pages/dashboard/pages/types.php
@@ -56,9 +56,9 @@ class Concrete5_Controller_Dashboard_Pages_Types extends DashboardBaseController
 		
 		if (!$ctName) {
 			$this->error->add(t("Name required."));
-	    } else if (preg_match('/[^0-9A-Z\.\!\&\(\)\-\_ ]/i', $ctName)) {
-	        $this->error->add(t('Page type names can only contain letters, numbers, spaces and the following symbols: !, &, (, ), -, _.'));
-	    }
+		} else if (preg_match('/[<>;{}?"`]/i', $ctName)) {
+			$this->error->add(t('Invalid characters in page type name.'));
+		}
 		
 		
 		if (!$valt->validate('update_page_type')) {
```

----
## web/concrete/core/controllers/single_pages/dashboard/pages/types/add.php
### web/concrete/core/controllers/single_pages/dashboard/pages/types/add.php

### Diff
```diff:
--- a/web/concrete/core/controllers/single_pages/dashboard/pages/types/add.php
+++ b/web/concrete/core/controllers/single_pages/dashboard/pages/types/add.php
@@ -23,8 +23,8 @@ class Concrete5_Controller_Dashboard_Pages_Types_Add extends DashboardBaseContro
 		
 		if (!$ctName) {
 			$this->error->add(t("Name required."));
-		} else if (preg_match('/[^0-9A-Z\.\!\&\(\)\-\_ ]/i', $ctName)) {
-			$this->error->add(t('Page type names can only contain letters, numbers, spaces and the following symbols: !, &, (, ), -, _.'));
+		} else if (preg_match('/[<>{};?"`]/i', $ctName)) {
+			$this->error->add(t('Invalid characters in page type name.'));
 		}
 
 		$valt = Loader::helper('validation/token');
```

----
## web/concrete/core/helpers/text.php
### web/concrete/core/helpers/text.php

### Diff
```diff:
--- a/web/concrete/core/helpers/text.php
+++ b/web/concrete/core/helpers/text.php
@@ -33,9 +33,9 @@ class Concrete5_Helper_Text {
 	 * @param string $handle
 	 * @return string $handle
 	 */
-	public function urlify($handle) {
+	public function urlify($handle, $maxlength = PAGE_PATH_SEGMENT_MAX_LENGTH) {
 		Loader::library('3rdparty/urlify');
-		$handle = URLify::filter($handle);
+		$handle = URLify::filter($handle, $maxlength);
 		return $handle;
 	}
 		
```

----
## web/concrete/core/libraries/page_cache/types/file.php
### web/concrete/core/libraries/page_cache/types/file.php

### Diff
```diff:
--- a/web/concrete/core/libraries/page_cache/types/file.php
+++ b/web/concrete/core/libraries/page_cache/types/file.php
@@ -27,7 +27,7 @@ class Concrete5_Library_FilePageCache extends PageCache {
 				$dir = DIR_FILES_PAGE_CACHE . '/' . $key[0] . '/' . $key[1] . '/' . $key[2];
 			}
 			if ($dir && (!is_dir($dir))) {
-				mkdir($dir, DIRECTORY_PERMISSIONS_MODE, true);
+				@mkdir($dir, DIRECTORY_PERMISSIONS_MODE, true);
 			}
 			$path = $dir . '/' . $filename;
 			return $path;
@@ -37,7 +37,7 @@ class Concrete5_Library_FilePageCache extends PageCache {
 	public function purgeByRecord(PageCacheRecord $rec) {
 		$file = $this->getCacheFile($rec);
 		if ($file && file_exists($file)) {
-			unlink($file);
+			@unlink($file);
 		}
 	}
 
@@ -49,14 +49,14 @@ class Concrete5_Library_FilePageCache extends PageCache {
 	public function purge(Page $c) {
 		$file = $this->getCacheFile($c);
 		if ($file && file_exists($file)) {
-			unlink($file);
+			@unlink($file);
 		}
 	}
 
 	public function set(Page $c, $content) {
 		if (!is_dir(DIR_FILES_PAGE_CACHE)) {
-			mkdir(DIR_FILES_PAGE_CACHE);
-			touch(DIR_FILES_PAGE_CACHE . '/index.html');
+			@mkdir(DIR_FILES_PAGE_CACHE);
+			@touch(DIR_FILES_PAGE_CACHE . '/index.html');
 		}
 
 		$lifetime = $c->getCollectionFullPageCachingLifetimeValue();
```

----
## web/concrete/core/models/page_theme.php
### web/concrete/core/models/page_theme.php

### Diff
```diff:
--- a/web/concrete/core/models/page_theme.php
+++ b/web/concrete/core/models/page_theme.php
@@ -266,7 +266,7 @@ class Concrete5_Model_PageTheme extends Object {
 	
 	public function parseStyleSheet($file, $styles = false) {
 		$env = Environment::get();
-		$themeRec = $env->getRecord(DIRNAME_THEMES . '/' . $this->getThemeHandle() . '/' . $file, $this->getPackageHandle());
+		$themeRec = $env->getUncachedRecord(DIRNAME_THEMES . '/' . $this->getThemeHandle() . '/' . $file, $this->getPackageHandle());
 		if ($themeRec->exists()) {
 			$fh = Loader::helper('file');
 			$contents = $fh->getContents($themeRec->file);
```

----
## web/concrete/js/ccm.app.js
### web/concrete/js/ccm.app.js

### Diff
```diff:
--- a/web/concrete/js/ccm.app.js
+++ b/web/concrete/js/ccm.app.js
@@ -1,5 +1,5 @@
 function ccmLayout(areaNameNumber,cvalID,layout_id,area,locked){this.layout_id=layout_id;this.cvalID=cvalID;this.locked=locked;this.area=area;this.areaNameNumber=areaNameNumber;this.init=function(){var e=this;this.layoutWrapper=$("#ccm-layout-wrapper-"+this.cvalID);this.ccmControls=this.layoutWrapper.find("#ccm-layout-controls-"+this.cvalID);this.ccmControls.get(0).layoutObj=this;this.ccmControls.mouseover(function(){e.dontUpdateTwins=0;e.highlightAreas(1)});this.ccmControls.mouseout(function(){e.moving||e.highlightAreas(0)});this.ccmControls.find(".ccm-layout-menu-button").click(function(t){e.optionsMenu(t)});this.gridSizing()};this.highlightAreas=function(e){var t=this.layoutWrapper.find(".ccm-add-block");e?t.addClass("ccm-layout-area-highlight"):t.removeClass("ccm-layout-area-highlight")};this.optionsMenu=function(e){ccm_hideMenus();e.stopPropagation();ccm_menuActivated=!0;var t=document.getElementById("ccm-layout-options-menu-"+this.cvalID);if(!t){el=document.createElement("DIV");el.id="ccm-layout-options-menu-"+this.cvalID;el.className="ccm-menu ccm-ui";el.style.display="none";document.body.appendChild(el);t=$(el);t.css("position","absolute");var n='<div class="popover"><div class="arrow"></div><div class="inner"><div class="content">';n+="<ul>";n+='<li><a onclick="ccm_hideMenus()" class="ccm-menu-icon ccm-icon-edit-menu" dialog-title="'+ccmi18n.editAreaLayout+'" dialog-modal="false" dialog-width="550" dialog-height="280" dialog-append-buttons="true" id="menuEditLayout'+this.cvalID+'" href="'+CCM_TOOLS_PATH+"/edit_area_popup.php?cID="+CCM_CID+"&arHandle="+encodeURIComponent(this.area)+"&layoutID="+this.layout_id+"&cvalID="+this.cvalID+'&atask=layout">'+ccmi18n.editAreaLayout+"</a></li>";n+='<li><a onclick="ccm_hideMenus()" href="javascript:void(0)" class="ccm-menu-icon ccm-icon-move-up" id="menuAreaLayoutMoveUp'+this.cvalID+'">'+ccmi18n.moveLayoutUp+"</a></li>";n+='<li><a onclick=
```

** Too large, sniped!! **

----
## web/concrete/js/ccm_app/legacy_dialog.js
### web/concrete/js/ccm_app/legacy_dialog.js

### Diff
```diff:
--- a/web/concrete/js/ccm_app/legacy_dialog.js
+++ b/web/concrete/js/ccm_app/legacy_dialog.js
@@ -100,8 +100,11 @@ jQuery.fn.dialog.open = function(obj) {
 		},*/
 		
 		'open': function() {
-			$("body").attr('data-last-overflow', $('body').css('overflow'));
-			$("body").css("overflow", "hidden");
+			var nd = $(".ui-dialog").length;
+			if (nd == 1) {
+				$("body").attr('data-last-overflow', $('body').css('overflow'));
+				$("body").css("overflow", "hidden");
+			}
 		},
 		'beforeClose': function() {
 			var nd = $(".ui-dialog").length;
```

----
## web/concrete/tools/pages/url_slug.php
### web/concrete/tools/pages/url_slug.php

### Diff
```diff:
--- a/web/concrete/tools/pages/url_slug.php
+++ b/web/concrete/tools/pages/url_slug.php
@@ -1,9 +1,8 @@
 <?
 defined('C5_EXECUTE') or die("Access Denied.");
 if (Loader::helper('validation/token')->validate('get_url_slug', $_REQUEST['token'])) {
-	Loader::library('3rdparty/urlify');
-	$name = URLify::filter($_REQUEST['name']);
-
+	$text = Loader::helper('text');
+	$name = $text->urlify($_REQUEST['name']);
 	$ret = Events::fire('on_page_urlify', $_REQUEST['name']);
 	if ($ret) {
   		$name = $ret;
```

----
