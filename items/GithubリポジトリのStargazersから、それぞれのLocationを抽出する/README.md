
自分がOSSとして公開しているChefのプラグイン、[knife-zero](https://github.com/higanworks/)っていうのには76のスターが付いている。
取り立てて多いというほどの数でもないのだろうが、Stargazersのほとんどがまるで縁もゆかりもないのだ。しかも最近国外が多い気もしている。

この、お互いどこの馬の骨ともわからん状態というのがわりと面白く感じたので、せっかくだからとロケーションを確認したくなった。

## Pythonで書いてみる

使ったモジュールは`github3.py`, Pythonは2.7.10。


```sg.py
import os
from github3 import login

## トークン使わないとAPI上限にすぐ引っかかる。
gh = login(token=os.environ['GH_TOKEN'])
repo = gh.repository('higanworks', 'knife-zero')

## iter_stargazers()がUserをイテレータの形で返してくれる。
for x in repo.iter_stargazers():
    print x, gh.user(x).location

```

## 実行してロケーションを抽出してみた

結局表記バラバラでつらいｗ

```
$ python ./sg.py 
tkuchiki None
libero18 Japan
DQNEO Japan
mohitsethi None
marcy-terui Sapporo City, Hokkaido, Japan
kimikimi714 None
tkak Tokyo
MasahiroSakoda Kanagawa Pref. Japan
takus Japan
kwilczynski Tokyo, Japan
nabeken Tokyo, Japan
deeeki Japan
bageljp Tokyo, Japan
shigeya Tokyo, Japan
blp1526 Tokyo
cl-lab-k Japan
knakayama Japan
wslash Japan
linyows FuckOka, Japan
SlyDen Lviv, Ukraine
eigo-s None
threetreeslight Tokyo
y13i Tokyo, Japan
bangbangshoot None
fprg None
sanemat Tokyo, Japan
fumikony None
dataferret Halifax, Canada
ikuwow Japan
rmoriz Munich, Germany
arosenhagen Darmstadt
sakazuki Tokyo, Japan
9gel None
teragino None
ispern Kagoshima
torounit Matsumoto, Nagano, Japan
Gascar-ShunT Tokyo
imura81gt None
cblunt Devon, UK
yukimamire None
gregf Maine
Azulinho Bristol
paulmoon None
inokappa None
ChrisLundquist Seattle, Wa
takeharu Tokyo
antage Russia
Chirul0 None
mitto Yokohama
runningman84 None
ssdns None
kyle-johnson Seattle, WA
ryandjurovich Melbourne, Australia
rollbrettler Berlin
adampats None
hirak Tokyo
patcon Waffles
k-nii0211 Tokyo
sawanoboly Kobe, Japan
ghempton Seattle, WA
mrjcleaver Toronto, ON, Canada
yukihariguchi None
hsbt Tokyo, Japan
azet *
k-yamada japan
daften Gent
pwelch Silver Spring, MD
dalpo Italy
h4ck3rm1k3 Hopewell, NJ
gretel Hamburg, Germany
expvictordamian None
guiferrpereira Porto
andrefreitas Porto and Madeira, Portugal
ruimashita None
caleb Rochester, NY
4148 Berkeley, CA
```

確定で日本じゃない(Japan,Tokyo,Noneを除く)のは1/3ってとこでしたね。アフリカと南極以外は一通りでてくる。

そんななか、一番驚いたのは **FuckOka**, Japan からお越しの方。
