[トップページに戻る](https://teamhimeh.github.io/simdev_intro/index.html)

\chapter{ケース・スタディー1　市道化防止}
本章では，ケース・スタディーとして市道化防止機能の開発に取り組む．読者は実際に手を動かして市道化防止機能の開発に取り組むことによって，大規模ソフトウェアの手探りかたや，頻出の処理であるファイルの読み書き，GUIウィンドウ作成などを習得することができる．
\section{本章のゴール}
simutransでは都市の発展により市内建築物が建設された場合，隣接する道路は市道にする仕組みがある．これは欧州の道路事情には非常にマッチしているが，大した歩車分離もナシに車が60km/hを超えるスピードで走る日本にはなじまない機能である．

\begin{figure}
  \centering
  \includegraphics[width=10cm]{images/市道化防止/1.png}
  \caption{市道化により残念なことになった高規格道路の図}
\end{figure}

そこで，本章ではこの市道化を防止する機能の開発を行う．しかし，単純に市道化の機能を完全にナシにしてしまうのでは趣がない．Simutransのシステムとして市道化が適切な役割を演じられるように，以下の2つの方法で市道化を制御しよう．

\begin{enumerate}[label={改造\arabic*},nolistsep,leftmargin=*]
    \item 制限速度が一定以上の道路は市道化しないようにする．例えば，70km/h制限の幹線道路は市道化するが，100km/h制限の高速道路は市道化しないようにする．閾値（制限何km/h以上で市道化を防止するか）はsimconf.tabおよび高度な設定から編集可能として，セーブデータに保存する．
    \item 道路1タイル単位で市道化を制御する．GUIウィンドウで市道化に関するフラグを設け，道路建設時にユーザーが選択できるようにする．ちょうどOTRPにおける道路の設定のイメージである．（図\ref{OTRP_road_config}）
\end{enumerate}

\begin{figure}
  \centering
  \includegraphics[width=5cm]{images/市道化防止/2.png}
  \caption{OTRPの道路設定ウィンドウ}
  \label{OTRP_road_config}
\end{figure}

「市道化防止」という一つの目的のためにわざわざ2つの異なるアプローチを取るのは冗長に感じられるかもしれない．しかし，この2つの実装方法を学ぶことで，読者が自分でsimutrans本体改造をするときに必要となるかなりの知識を得ることができる．ぜひ，自分で手を動かし，じっくり考えながらこのチュートリアルを進めていってほしい．

\subsection*{サンプルコードについて}
先に挙げた2つのアプローチについて，\ref{改造1}節では1つ目の方法で，\ref{改造2}節では2つ目の方法で改造を行う．本書では読者の利便のため，この2つの改造それぞれについてサンプルコードを提供する．サンプルコードはsimutrans standard nightly r8635を改造したものである．サンプルコードは以下のURLから参照できる．
\begin{description}
  \item[改造1（\ref{改造1}節）] \url{https://github.com/teamhimeh/simutrans/tree/intro_of_sim_dev_1}
  \item[改造2（\ref{改造2}節）] \url{https://github.com/teamhimeh/simutrans/tree/intro_of_sim_dev_2}
\end{description}

\section{市道化処理の場所特定}
さて，実装方針は決まったわけであるが，そもそも「どこ」に市道化処理が書かれているのだろうか．市道化処理が書かれている場所を特定できれば，そこを変更することで市道化の挙動を変更できることになる．

100行ほどの小さなプログラムであれば，main関数からがんばって処理の流れを追うことでプログラム全体を理解することは難しくない．しかし，simutransのコードベースは数万行（数十万行？）と巨大である．よって，main関数から全てをたどっていくなんてことは到底不可能である．

simutransのような大規模なソフトウェアに手を加える時，一般には下記の手順を踏む．
\begin{itembox}[c]{大規模ソフトウェアの手探りかた}
  \begin{enumerate}
    \item どのファイルをいじれば良いのか見つける．すなはち，改造したい機能が書かれたファイルを見つける．
    \item どのclassが関与しているのか見つける．
    \item 機能を改造するために必要な関数をclass内から見つける．
    \item その関数が行っている処理を理解する．
    \item 実際にその関数を書きかえてみる．
  \end{enumerate}
\end{itembox}

それでは，この手順に従って，改造するファイル・クラスを見つけるところから始めよう．

\subsection{ファイル・クラスを見つける}
simutransはオブジェクト指向で書かれている．すなはち，オブジェクト同士が命令しあって巨大なシステムができあがっている．したがって，編集したいファイル・クラスを見つける基本的な方針は
\begin{screen}
  \begin{center}
    「\underline{\bf{誰が}}その処理をやっているのか」で見当をつける
  \end{center}
\end{screen}
となる．

いくつか例を与えるので，テキストエディタの横断検索機能やエクスプローラーの検索機能などを使って実際に検索しながら追ってみてほしい．例えば，飛行機の着陸待機中の挙動を変えたいとする．この時，着陸待機中の動きは「飛行機」がやっている．つまり「飛行機」を定義するクラスを探せばよい．simutransソースフォルダを眺めていると「vehicle」フォルダが見つかるので，ここにありそうである．さらに，飛行機なので「air」で検索をかけてみる．すると，vehicle/simvehicle.hの中に「air\_vehicle\_t」というクラスが見つかった．

別の例として，斜め線路上に駅を設置できるようにしたいとする．この時，「誰」が駅を設置するのかと言われれば，プレイヤーである．しかし，プレイヤーは駅建設ツールを使って駅を設置している．すなはち，駅を設置しているのは「駅建設ツール」である．ツールのクラスかぁということでsimutransソースフォルダを眺めていると，simtool.hというファイルが見つかった．そこで，simtool.hを開いてみると，大変巨大なファイルである．駅に関連するツールなので「station」で検索してみると，「tool\_build\_station\_t」というクラスが見つかった．\footnote{別のたどり方として，「ツール」に着目する方法がある．改造したいツールが既にわかっている時，menuconfを参照すると割り当てられたキーなどからそのツールの番号を知ることができる．simmenu.hに並ぶツールのenumの並び順はmenuconfでのツール番号と対応しているので，改造するツールに対応するenumを知ることができる．目的のツールのenumをsimtool.hで検索すると，目的のツールのクラスに到達することができる．}

このように，\underline{\bf{「実行主体から階層をたどる」+「それっぽいキーワードで検索をかける」}}のあわせ技でファイル・クラスを見つけるのが定石である．実行主体を想定してファイル階層をたどるときは図\ref{directory}が大きな助けになるだろう．

それでは，今回の市道化防止機能の場合はどのファイル・クラスを編集すれば良いのだろうか？市道化は都市内に市内建築物が建設or建て替えられたときに発生する．市内建築物を配置しているのは「都市」であるから，市道化を行っているのも「都市」であると考えられる．このような視点のもとsimutransソースフォルダを眺めていると，simcity.h/simcity.ccというファイルが見つかる．クラスの機能や性質を知るには，まずはヘッダファイルを読めばよい．simcity.hを開くと，しばらくincludeや各種宣言がつづいたあとクラスの定義が始まる．（コード\ref{simcity_class_def}）

\begin{lstlisting}[caption=simcity.h抜粋　class stadt\_t定義の冒頭, style=customC, label=simcity_class_def]
/**
 * Die Objecte der Klasse stadt_t bilden die Staedte in Simu. Sie
 * wachsen automatisch.
 * @author Hj. Malthaner
 */
class stadt_t
{
\end{lstlisting}

ドイツ語のコメントを英語にgoogle翻訳すると「The objects of class city\_t are the cities in Simu. They grow automatically.」である．\verb|stadt_t|は紛れもなくsimutransにおける都市を定義するクラスであるとわかる．なお，英語翻訳ではなく日本語翻訳を使用すると「
city\_tクラスのオブジェクトはSimuの都市です．彼らは自動的に成長する．」と出力される．日本語訳でも読めなくはないが，英語訳のほうが圧倒的に意味を取りやすいことがわかるだろう．ちなみに，「stadt」という単語はドイツ語で都市という意味である．

ともかく，市道化の処理が行われているクラスはsimcity.hに記述された\verb|stadt_t|であるようだということがわかった．慣れないうちは一発でファイル・クラスを特定することは難しいので，いくつかアテをつけてヘッダファイルを読んで，そこにある変数名や関数名から該当クラス候補を絞り込んでいくことになる．経験を積むと，どの処理がどのファイル・クラスにかかれているのかおおよそ一発で分かるようになる．

\subsection{関数を見つける}
市道化処理は\verb|stadt_t|のどの関数にかかれているのだろうか．ここでの基本的な方針は
\begin{screen}
  \begin{center}
    「\underline{\bf{ヘッダファイル}}のコメント・関数名で見当をつける
  \end{center}
\end{screen}
である．まずは，ヘッダファイルを流し読みしてそのクラスがどんなパラメータを持ち，どんな仕事をするのか大雑把に理解する．この時点では\bf 実装ファイルは読まない． \rm 

コード\ref{header_reading}はsimcity.hの一部を抜粋したものである．ヘッダファイルではコメントと共に関数が宣言されている．ここでは\verb|stadt_t|に\verb|calc_growth()|という関数が定義されていること，その関数は都市の成長率を計算する役割を持つということだけ理解すれば十分である．その関数がどのように都市の成長率を計算しているか知る必要はない．実装ファイルではなくヘッダファイルをもっぱら読むのはこういう意味である．都市の成長率計算の詳細を知る必要がある時，もしくは処理の中身を読まないと関数の役割がいまいちよくわからない時は，実装ファイルを読めばよい．

\begin{lstlisting}[caption=simcity.h抜粋2　関数の宣言, style=customC, label=header_reading]
/**
 * Recalculates city borders (after loading and deletion).
 * @warning Do not call this during multithreaded loading!
 */
void recalc_city_size();

// calculates the growth rate for next growth_interval using all the different indicators
void calc_growth();
\end{lstlisting}

さて，市道化を行っている関数を見つけるにはどうすればいいのだろうか．「市道」であるから，適当に「street」で検索をかけてやればそれなりのものが見つかるかもしれない．しかし，simcity.hで「street」の文字列検索をかけても残念ながら何もヒットしない．このように探してもよくわからない場合は，一人で悩むより諦めて本家フォーラムに聞きに行くのが一番早かったりする．本家フォーラムに質問をするときは，同じ質問が既にされていないか検索をかけるのを忘れないようにしよう．

さて，ヘッダファイル内をいくら探しても市道化を行う関数は見つからない．探しているクラスが間違っているのだろうか？そうではない．実は今回の場合，simcity.ccに\verb|update_city_street()|という関数があり，これが市道化処理の本体である．コード\ref{upd_str_head}は\verb|update_city_street()|の冒頭部分である．
\newpage
\begin{lstlisting}[caption=市道化処理関数はsimcity.ccにあるupdate\_city\_street(), style=customC, label=upd_str_head]
// updates one surrounding road with current city road
bool update_city_street(koord pos)
{
	const way_desc_t* cr = world()->get_city_road();
	for(  int i=0;  i<8;  i++  ) {
		if(  grund_t *gr = world()->lookup_kartenboden(pos+neighbors[i])  ) {
			if(  weg_t* const weg = gr->get_weg(road_wt)  ) {
\end{lstlisting}

ところで，simcity.ccでは，\verb|stadt_t|に属している他の関数の冒頭はコード\ref{func_def_in_cc}のようなスタイルで始まっている．関数名の前に「\verb|stadt_t::|」がついていることがわかる．「\verb|stadt_t::|」はクラス\verb|stadt_t|に属している関数の宣言であることを表す．

\begin{lstlisting}[caption=.ccファイル内では関数名の前にクラス名がつく, style=customC, label=func_def_in_cc]
void stadt_t::check_bau_factory(bool new_town)
{
	uint32 const inc = welt->get_settings().get_industry_increase_every();
\end{lstlisting}

ところが，コード\ref{upd_str_head}にあるように\verb|update_city_street()|には「\verb|stadt_t::|」がついていない．すなはち，これはクラスに属さない関数なのである．クラスに属していないのでヘッダファイルをいくら探しても記述がなかった．なお，simcity.ccで\verb|update_city_street()|で文字列検索をかけると，この関数は\verb|stadt_t::renovate_city_building|と\verb|stadt_t::build_city_building|で呼ばれていることがわかる．このことから，市道化処理は市内建築物が建てられた時および建て替えられた時に呼ばれることがわかる．この2つはクラス\verb|stadt_t|に属する関数であり，ヘッダファイルにも書かれている．

\subsection{市道化処理を理解する}
それでは，simcity.ccの\verb|update_city_street()|を読んで市道化処理を理解しよう．コード\ref{upd_city_street}では，できるだけ丁寧に日本語でのコメントを付した．お手元のソースコードと見比べながら読んでほしい．

\begin{lstlisting}[caption=市道化処理が書かれているupdate\_city\_street(), style=customC, label=upd_city_street]
// updates one surrounding road with current city road
// 引数:建設or建て替えされた市内建築物の座標
bool update_city_street(koord pos)
{
  // 市道のdescriptor（pak情報）を取得しcrに代入
	const way_desc_t* cr = world()->get_city_road();
  
  // 与えられた建物座標に隣接する8マスについて処理する
	for(  int i=0;  i<8;  i++  ) {
    // 座標を指定して地面を取得し，grに代入
		if(  grund_t *gr = world()->lookup_kartenboden(pos+neighbors[i])  ) {
      // gr（地面）から道路を取得し，wegに代入
			if(  weg_t* const weg = gr->get_weg(road_wt)  ) {
				// Check if any changes are needed.
        // 道路に歩道がないOR市道じゃない→市道化！
				if(  !weg->hat_gehweg()  ||  weg->get_desc() != cr  ) {
          // spは道路のもともとの所有者を保持
					player_t *sp = weg->get_owner();
					if(  sp  ){
            // この道路はプレイヤーに属している．コストの計算をする．
						player_t::add_maintenance(sp, -weg->get_desc()->get_maintenance(), road_wt);
            // 道路の所有者をNULLで市道化
						weg->set_owner(NULL); // make public
					}
          // 歩道をつける
					weg->set_gehweg(true);
          // 道路のdesc（pak情報）を市道のやつに差し替え
					weg->set_desc(cr);
          // 描画計算を指示
					gr->calc_image();
          // ミニマップも再描画
					reliefkarte_t::get_karte()->calc_map_pixel(pos+neighbors[i]);
					return true;	// update only one road per renovation
				}
			}
		}
	}
	return false;
}
\end{lstlisting}

このコードでは，建物座標のまわり8タイルについて市道化が必要な道路があるか検査し，あればdescriptorを差し替えて（つまり100km/h幹線道路を50km/h市内道路にする）歩道をつけてownerをNULLにしている．これが市道化の一連の流れである．ここからわかるように，コード\ref{upd_city_street}のfor文の中身を全部消せば市道化は起こらなくなる．実際に実験して確認してほしい．

\section{地面タイル上のオブジェクト配置構造}

ところで，コード\ref{upd_city_street}を見ていると\verb|grund_t|やら\verb|lookup_kartenboden|やら\verb|get_weg|やらと見慣れない文字列が登場してくる．これらは大変使用頻度の高い関数群であるので，簡単に整理しておこう．

\begin{figure}
  \centering
  \includegraphics[width=10cm]{images/市道化防止/6.png}
  \caption{道路タイルのオブジェクト}
  \label{obj_on_tile1}
\end{figure}

図\ref{obj_on_tile1}は，simutransにおける道路がひかれたある1タイルの様子を表している．まず，地面タイルオブジェクトがある．本当は図\ref{inh_grund}で示したようにタイルオブジェクトは\verb|grund_t|の子クラスの形で定義されるのだが，通常はべつに高架のタイルだろうと地面のタイルだろうと大して差はないので親クラスの\verb|grund_t|の形で包括的に扱っている．地面の上には「道」（weg）が載っている．これは道路の場合もあるし，鉄軌道，滑走路の場合もある．道路が載っている場合はさらに市電軌道も併存可能である．そして，そこに車両が走っている．この場合は，車（road vehicle）が走っている．

ところが，図\ref{obj_on_tile1}のイメージはオブジェクトの管理関係としてあまり正しくない．正しいのは図\ref{obj_on_tile2}の方である．道も車両も（そして実は建物も）\verb|obj_t|というクラスを祖先に持っている．そして，地面タイルはタイル上にのっているものを包括的に「obj」として管理している．実際，boden/grund.hにはタイルに「ぶらさがっている」オブジェクトたちを管理する変数がある．

\begin{lstlisting}[caption=grund\_tのobjlist, style=customC]
/**
　* List of objects on this tile
　* Pointer (changes occasionally) + 8 bits + 8 bits (changes often)
　*/
objlist_t objlist;
\end{lstlisting}

\begin{figure}
  \centering
  \includegraphics[width=8cm]{images/市道化防止/3.png}
  \caption{道路も車両もobjとして地面タイルに管理されている}
  \label{obj_on_tile2}
\end{figure}

ここで，所望の座標の道オブジェクトを取得する手順を整理しておこう．コード\ref{get_way_from_pos}は与えられた座標（\verb|pos_3d|または\verb|pos_2d|）にある道路オブジェクトを変数\verb|weg|に取得するコードである．与えられた座標に対応する地面オブジェクトを取得し，その地面オブジェクトに対して道路オブジェクトを要求する．

\begin{lstlisting}[caption=座標から道オブジェクトを取得する, style=customC, label=get_way_from_pos]
// koord3dは3次元座標，koordは2次元座標のクラス名である
koord3d pos_3d = koord3d(114,51,4);
koord pos_2d = koord(8,10);

// 地面オブジェクトを取得する
grund_t* gr;
// 3次元座標の場合はlookupを使う
gr = world()->lookup(pos_3d);
// 2次元座標の場合はlookup_kartenbodenを使う
gr = world()->lookup_kartenboden(pos_2d);

// 道（線路，道路，滑走路など）を取得
// 三項演算子でgrがNULLでないことを必ず確認する
weg_t* weg = gr ? gr->get_weg(road_wt) : NULL;
\end{lstlisting}

\verb|world()|という関数はどこから出てくるんだという話になるが，これはsimworld.hで提供されているpublicな関数である．simworld.hは大抵の場合既にincludeされているし，もしそうでない場合で利用したいならsimworld.hをincludeすればよい．\verb|world()|で得られるのは文字通り「世界」である．

本体改造初心者にありがちなミスの一つが，null pointer access（通称ヌルポ），すなはち，NULLが代入されているポインタ変数を呼んでしまうことである．\verb|lookup|や\verb|lookup_kartenboden|は有効でない座標を引数にわたすとNULLを返すため，結果を使うときは必ずNULLチェックをしなければならない．

ちなみに，座標から車両オブジェクトを取得する手順はコード\ref{get_vehicle_from_pos}のようになる．地面オブジェクトを取得し，その地面オブジェクトにぶら下がっているオブジェクト配列を要求する．\verb|get_weg()|のような便利な関数はないので，\verb|objlist|の要素をfor文で一つずつ検査して車両オブジェクトを検出する．こちらも改造内容によってはよく使うので押さえておこう．

\begin{lstlisting}[caption=座標から車両オブジェクトを取得する, style=customC, label=get_vehicle_from_pos]
koord3d pos_3d = koord3d(8,9,3);
grund_t* gr;
gr = world()->lookup(pos_3d);
// grのNULLチェックを忘れないようにしよう
if(  !gr  ) {
  return false;
}

for(uint8 idx=1;  idx<(volatile uint8)gr->get_top();  idx++) {
  if(  vehicle_base_t* const v = obj_cast<vehicle_base_t>(gr->obj_bei(idx))  ) {
    // vに車両オブジェクトが代入されているので処理をする
  }
}
\end{lstlisting}

\section{バグ修正の基本的な方針}
この先の章で実際にソースコードを改変して改造に取り組むことになる．本書の通りコードを改変していけば期待した結果が得られるはず，なのであるが，不具合に悩まされることなしに最後にたどり着くことは極めてまれである．不具合の原因はタイポであるかもしれないし，編集箇所を間違えたからかもしれないし，環境的な問題かもしれない．いずれにせよ，これから直面するであろう不具合に自力で対処しなければならないのである．そこで，この章では不具合修正の方針を簡単に説明する．

\subsection{デバッグの基本的方針}
不具合に遭遇したときは以下の流れで立ち向かうことになる．
\begin{enumerate}
  \item バグを確実に再現させる条件を作る．
  \item 何が起こっているのかの詳細な情報をprintfやデバッガを駆使して収集する．情報をもとに，問題がありそうな場所を絞り込んでいく．
  \item 問題箇所を修正し，正常に動くか入念にテストする．
\end{enumerate}
バグを確実に再現させる条件を作るというのは，世の中には「たまに発現する」バグというのが存在するからである．この場合，原因箇所を探る前に「いつそのバグは起こるのか」を特定せねばならない．そうでないと情報収集ができないからである．

デバッグで一番難しいことは，問題箇所を特定することである．問題箇所の特定は起こっていることの情報（例えばその時の変数の値）を元にして行う．そこで，以下の節ではいくつかの情報収集の手法について説明する．

\subsection{printfデバッグ}
原始的なデバッグ方法の一つは，気になる変数の値をprintf関数でコンソールに表示させることである．ここでは例として，市道化が行われた道路の座標を表示させることを考える．

市道化を行う関数はsimcity.ccに記述された\verb|update_city_street()|であった．ここに，コード\ref{printf_ucs}のようにprintf文を埋め込む．実際に市道化が行われたときのみ座標を表示したいので，printf文は関数の冒頭ではなく10行目に埋め込んである．

\begin{lstlisting}[caption=市道化が行われた道路の座標を表示（simcity.cc）, style=customC, label=printf_ucs]
bool update_city_street(koord pos)
{
	const way_desc_t* cr = world()->get_city_road();
  
	for(  int i=0;  i<8;  i++  ) {
		if(  grund_t *gr = world()->lookup_kartenboden(pos+neighbors[i])  ) {
			if(  weg_t* const weg = gr->get_weg(road_wt)  ) {
				// Check if any changes are needed.
				if(  !weg->hat_gehweg()  ||  weg->get_desc() != cr  ) {
          printf("update_city_street():%d,%d,%d \n", pos.x, pos.y, pos.z); 
          // \nは改行コード
					player_t *sp = weg->get_owner();
					if(  sp  ){
						player_t::add_maintenance(sp, -weg->get_desc()->get_maintenance(), road_wt);
						weg->set_owner(NULL); // make public
					}
（以下省略）
\end{lstlisting}

このコードを実行し，人口増加ツールなどで適当な道路を市道化させると，
\begin{lstlisting}[caption=市道化された座標の出力, style=bash]
update_city_street():8,9,3
update_city_street():810,191,9
\end{lstlisting}
といった具合に，市道化が行われた座標がコンソールに出力される．

コード\ref{printf_ucs}の10行目では，三次元座標をprintf出力させるために，以下のコードを用いている．三次元座標変数\verb|pos|について，x,y,z成分をそれぞれ出力している．
\begin{lstlisting}[caption=座標のprintf, style=customC]
printf("update_city_street():%d,%d,%d \n", pos.x, pos.y, pos.z);
\end{lstlisting}

実は，クラス\verb|koord3d|はデバッグのために\verb|get_str()|関数を提供している．この関数は座標の文字列を返すので，コード\ref{get_str_deprecated}のように記述が簡潔になる．
\begin{lstlisting}[caption=get\_str()関数．使用は推奨されない, style=customC, label=get_str_deprecated]
printf("update_city_street():%s \n", pos.get_str());
\end{lstlisting}
しかし，座標を出力するために\verb|get_str()|を用いることは推奨されない．\verb|get_str()|はchar型のポインタを返すが，そのポインタが指す先はstaticなchar配列である．これによって，\verb|get_str()|は誤った出力を返すことがあるので，デバッグのときは面倒でもx,y,zをそれぞれ出力すべきである．

素のprintf文はデバッグが終わったら取り除かなければならない．しかし，場合によってはデバッグ用の出力をコードの中に恒久的に残したい場合がある．Simutransではデバッグ出力の方法が用意されているので，正式にはこちらを使うべきである．デバッグ出力の書式は以下の通りである．
\begin{lstlisting}[caption=デバッグ出力構文, style=customC, label=fatal_example]
dbg->fatal( "class_name::func_name()", "Error Message!" );
\end{lstlisting}
第一引数にはメッセージ発行場所，第二引数にはメッセージの内容を記すのが通例である．デバッグメッセージにはランクがあり，重要度が低い順にmessage，debug，important，warning，error，fatalがある．コード\ref{fatal_example}はfatalメッセージの例である．fatalメッセージが発行されるとsimutransはエラーメッセージを出して動作を停止する．詳しくはutils/log.hおよびsimdebug.hを参照されたい．

\subsection{デバッガを使った情報収集}
printfデバッグは簡単である反面，高度な情報収集は困難でかつprintfを埋め込むごとに再コンパイルが必要という欠点がある．本書では環境構築の章でsimutransをgdbというデバッガ上で動作させるよう案内した．デバッガを使うことで，再コンパイルいらずで以下のような情報収集が可能である．
\subsubsection{ブレークポイント}
ブレークポイントを設定すると，そこで実行が一時停止する．一時停止させれば後で述べるバックトレースを取ったり変数の値を表示したりできる．ブレークポイントの設定はコード\ref{set_breakpoint}のようにする．
\begin{lstlisting}[caption=ブレークポイントの設定, style=bash, label=set_breakpoint]
(gdb) b ファイル名:行番号
例) (gdb) b simvehicle.cc:1919
\end{lstlisting}
\subsubsection{バックトレース}
現在実行してる行が「誰によって」呼ばれたかを表示する．出力は階層的になる．セグフォでクラッシュするバグの対応にはコレが便利．
\begin{lstlisting}[caption=バックトレースの出力, style=bash]
(gdb) bt
\end{lstlisting}
\subsubsection{変数表示}
現在の変数の値を表示できる．
\begin{lstlisting}[caption=変数の値の出力, style=bash]
(gdb) p 変数名
\end{lstlisting}

デバッガには他にも高度な情報収集コマンドがたくさんある．ぜひ一度は調べて使いこなせるようになってほしい．

\section{改造1: 一定制限速度以上の道路は市道化しない}
\label{改造1}
それでは，1つめのやり方で改造に取り掛かるとしよう．後ろにもう一つの改造が控えているので，gitで適当にブランチを切ってから作業することを勧める．コマンドラインではコード\ref{createNewBranch}のようにすれば，「speed\_threshold」という名前のブランチを作成し，同時にそのブランチに切り替えることができる．

\begin{lstlisting}[caption=gitでブランチを切る, style=bash, label=createNewBranch]
git checkout -b speed_threshold
\end{lstlisting}

今から行う改造は，「一定制限速度以上の道路であれば市道化しない」というものである．速度の閾値はユーザーが自由に決められるようにすべきである．しかし，いきなり一般論から入ると難しいので，まずは60km/h以上の道路は市道化しないことにしたい．どうすればいいのだろうか？考えて手を動かして実験してみよう．\\
\dotfill

一例として，\verb|update_city_street|を以下のように編集してみる．
\begin{lstlisting}[caption=60km/h以上の道路は市道化しない, style=customC, label=threshold_60]
bool update_city_street(koord pos)
{
	const way_desc_t* cr = world()->get_city_road();
	for(  int i=0;  i<8;  i++  ) {
		if(  grund_t *gr = world()->lookup_kartenboden(pos+neighbors[i])  ) {
			if(  weg_t* const weg = gr->get_weg(road_wt)  ) {
				// Check if any changes are needed.
				if(  (!weg->hat_gehweg()  ||  weg->get_desc() != cr)  &&  weg->get_max_speed()<60  ) {
          // 市道化する
					player_t *sp = weg->get_owner();
					if(  sp  ){
						player_t::add_maintenance(sp, -weg->get_desc()->get_maintenance(), road_wt);
						weg->set_owner(NULL); // make public
					}
					weg->set_gehweg(true);
					weg->set_desc(cr);
（以下省略）
\end{lstlisting}
コード\ref{threshold_60}の8行目に注目すると，if文の条件に「\verb|weg->get_max_speed()<60|」が追加された．これで，道路の制限速度が60km/hより小さいときのみ市道化が実行される．

\subsection{settings\_t}
\label{subsec_settings_t}
それでは，この「60km/h」を一般化しよう．simutransではゲームのパラメータはセーブデータ内に保存することになっている．パラメータ設定のためにsimuconf.tabを参照するのは初回起動時だけというのが原則である．これは，ネットワークゲームにおいてsimuconf.tabの違いが挙動に影響を及ぼすことを防ぐためである．

simutransではこの「setting」を扱うクラスが用意されている．それはdataobj/settings.hに書かれている\verb|settings_t|である．お手元の環境でdataobj/settings.hを開いてほしい．

settings.hには「高度な設定」で見覚えのあるパラメータたちがたくさん並んでいることがわかる．\verb|settings_t|はパラメータを専門に扱うクラスである．ヘッダファイルは各パラメータに対してコード\ref{settings_t}のような構成で書かれている．

\begin{lstlisting}[caption=各パラメータに変数宣言・getter・setterがある, style=customC, label=settings_t]
private:
  sint32 traffic_level;
public:
  sint32 get_traffic_level() const { return traffic_level; }
  void set_traffic_level(sint32 l) { traffic_level=l; }
\end{lstlisting}
パラメータがprivate属性で用意され，public属性でそのgetter・setter関数が用意されている．変数自体をprivateにして外部から変数にアクセスする時はgetter・setterを使うという仕組みは，オブジェクト指向プログラミングの定石である．

それでは，ここにパラメータを追記しよう．「制限XXキロ以上なら市道化しない」という値なので，\verb|max_cityroad_speed|という名前にしてみる．すると，\verb|settings_t|のヘッダファイルにコード\ref{mod_settings_t}のように書き足せばいいことになる．

\begin{lstlisting}[caption=max\_cityroad\_speedの定義, style=customC, label=mod_settings_t]
private:
  uint16 max_cityroad_speed;
public:
  void set_max_cityroad_speed(uint16 l) {max_cityroad_speed=l;}
  uint16 get_max_cityroad_speed() const {return max_cityroad_speed;}
\end{lstlisting}
\verb|weg_t|において\verb|max_speed|がuint16で定義されているのでここでも変数サイズをuint16とした．そして，simcity.ccの\verb|update_city_street()|においてコード\ref{threshold_gen}のように「60km/h」を\verb|max_cityroad_speed|に置き換えてあげればよい．

\begin{lstlisting}[caption=simcity.cc　閾値速度の一般化, style=customC, label=threshold_gen]
bool update_city_street(koord pos)
{
	const way_desc_t* cr = world()->get_city_road();
	for(  int i=0;  i<8;  i++  ) {
		if(  grund_t *gr = world()->lookup_kartenboden(pos+neighbors[i])  ) {
			if(  weg_t* const weg = gr->get_weg(road_wt)  ) {
				// Check if any changes are needed.
        uint16 threshold = world()->get_settings().get_max_cityroad_speed();
				if(  (!weg->hat_gehweg()  ||  weg->get_desc() != cr)  &&  weg->get_max_speed()<threshold  ) {
          // 市道化する
					player_t *sp = weg->get_owner();
（以下省略）
\end{lstlisting}
8行目で閾値速度をローカル変数\verb|threshold|として用意し，9行目でそれを使っている．\verb|world()->get_settings()|で\verb|settings_t|オブジェクトを取得できる．ここで取得されるのは参照型なので，メンバ関数にはアロー演算子ではなくドットでアクセスする．

さて，未だ2つの問題が残っている．
\begin{itemize}
  \item \verb|max_cityroad_speed|をユーザーが設定する手段がない．
  \item \verb|max_cityroad_speed|がセーブデータに保存されていない．
\end{itemize}
1つめの問題は次節で対処するとして，2つめの問題を片付けてしまおう．

先ほど編集したのはsettings.hであった．ディレクトリをよく見るとsettings.ccなるファイルも存在する．今からsettings.ccで行う作業は「初期化」「simuconfのパース」「データのセーブ」である．お手元のdataobj/settings.ccを開いて一緒に作業を進めていこう．

まず\verb|settings_t|のコンストラクタである\verb|settings_t::settings_t()|で初期化を行う．コード\ref{settings_cc_1}の7行目がそれである．適当にコンストラクタの末尾にでも書き足しておけばよかろう．（r8549現在ではdataobj/settings.ccの300行目前後）
\begin{lstlisting}[caption=max\_cityroad\_speedの初期化 in settings\_t::settings\_t(), style=customC, label=settings_cc_1]
  random_counter = 0;	// will be set when actually saving
	frames_per_second = 20;
	frames_per_step = 4;
	server_frames_ahead = 4;
  
  // テキトーに60km/hで初期化すればいいかな
  max_cityroad_speed = 60;
}
\end{lstlisting}

次に，simuconf.tabを読み込む．これは\verb|settings_t::parse_simuconf()|が行うので，その中に追記してあげる．これも関数末尾に近い位置に追記すればよい．（コード\ref{settings_cc_2}の5行目）ちなみに，\verb|contents.get_int()|の第二引数はsimuconf.tabから該当パラメータが見つからなかったときの初期値である．\verb|max_cityroad_speed|はコンストラクタで既に初期化してあるので第二引数には設定するパラメータ自身を使う．
\begin{lstlisting}[caption=simuconf.tabの読み込み in settings\_t::parse\_simuconf(), style=customC, label=settings_cc_2]
  max_ship_convoi_length = contents.get_int("max_ship_convoi_length",max_ship_convoi_length);
  max_air_convoi_length = contents.get_int("max_air_convoi_length",max_air_convoi_length);

  // simuconf.tab内の"max_cityroad_speed"という項目を読む
  max_cityroad_speed = contents.get_int("max_cityroad_speed", max_cityroad_speed);

  // Default pak file path
  objfilename = ltrim(contents.get_string("pak_file_path", "" ) );
  printf("Reading simuconf.tab successful!\n" );
}
\end{lstlisting}

これで，simuconf.tabからパラメータをセットできるようになった．simuconf.tabに「\verb|max_cityroad_speed=80|」と追記して，起動してみよう．制限速度80km/h以上の道路が市道化しなくなっていることを確認してほしい．

以上で「一定制限速度以上の道路は市道化しない」改造は達成されたように見える．実際，私的な範囲で使う分にはあまり問題はない．しかし，本家統合などが目的の場合は，本節の冒頭で述べた理由によりパラメータ\verb|max_cityroad_speed|をセーブデータ内に保存する必要がある．そこで，次節ではセーブデータの読み書きを行う．

\subsection{セーブデータの読み書き}
\label{セーブデータの読み書き}
セーブデータの読み書きは\verb|settings_t::rdwr()|で行う．なお，\verb|grund_t|や\verb|weg_t|や\verb|vehicle_t|などといったあらゆるクラスに\verb|rdwr()|関数は用意されており，そこでデータの読み出し・保存を行うことになっている．ロード・セーブで別々の関数が用意されているのではなく，ロード時・セーブ時ともに\verb|rdwr()|関数が呼ばれることに注意されたい．ここで，\verb|settings_t|の\verb|rdwr()|の一部を覗いてみよう．コード\ref{rdwr}はdataobj/settings.ccの\verb|settings_t::rdwr()|の一部であり，\verb|settings_t|の\verb|bonus_basefactor|という変数の値を読み書きしている．
\begin{lstlisting}[caption=rdwr()の基本書式, style=customC, label=rdwr]
if(  file->get_version()>=111002  ) {
  file->rdwr_long( bonus_basefactor );
}
else if(  file->is_loading()  ) {
  // 古いバージョンなので読み出しせず125という値を代入する．
  bonus_basefactor = 125;
}
\end{lstlisting}
コード\ref{rdwr}の1行目でバージョンチェックをしている．この場合，バージョンが111002以上であれば読み書き，それ以前であれば読み書きせず値を代入している．2行目にある\verb|rdwr_long()|は32bitのパラメータを読み書きする場合に使い，64bitの場合は\verb|rdwr_longlong()|，16bitの場合は\verb|rdwr_short()|，8bitの場合は\verb|rdwr_byte()|を使う．bool値の場合は\verb|rdwr_bool()|を使う．ここで引数に渡しているのは参照であって（\verb|loadsave_t|のヘッダファイルを読むとわかる．），読み出しの際は引数として渡した変数に値が格納され，書き出しの際は引数として渡した値がデータに書き込まれる．simutransではセーブデータ内に変数がindexナシに順番に格納されるので，ここで\underline{読み書きする順番に誤りがあると処理に支障をきたす}ことに注意されたい．例えば，古いデータを読むときに誤って新しいバージョンでしか定義されていないパラメータを読み出そうとしたとする．このとき，「パラメータが見つからないからエラー」ではなく「読み出す順番がズレて処理続行不能」となるのである．

ところで，セーブデータのバージョンはどこで定義されているのだろうか．その答えはsimversion.hにある．コード\ref{simversion}はsimversion.h 13行目からのの抜粋である．

\begin{lstlisting}[caption=simversion.h抜粋（13行めから）, style=customC, label=simversion]
#define SIM_VERSION_MAJOR 120
#define SIM_VERSION_MINOR   4
#define SIM_VERSION_PATCH   1
#define SIM_VERSION_BUILD SIM_BUILD_NIGHTLY

// Beware: SAVEGAME minor is often ahead of version minor when there were patches.
// ==> These have no direct connection at all!
#define SIM_SAVE_MINOR      7
#define SIM_SERVER_MINOR    7
// NOTE: increment before next release to enable save/load of new features

#define MAKEOBJ_VERSION "60.1"
\end{lstlisting}

\verb|rdwr()|で扱うデータのバージョン番号は以下のように計算される．

\begin{itembox}[c]{セーブデータバージョン番号の計算式}
  \begin{center}
    バージョン番号 = \verb|SIM_VERSION_MAJOR×1000 + SIM_SAVE_MINOR|
  \end{center}
\end{itembox}

各種バージョン番号がコード\ref{simversion}のようになっているならば，その状態でデータを保存するとバージョン番号は「120*1000+7=120007」と計算される．読み書きするパラメータを新しく増やすときは\verb|SIM_SAVE_MINOR|の値を増やした上で\verb|rdwr()|で適切にバージョンによる場合分けをしなければならない．

それでは，\verb|max_cityroad_speed|のデータを読み書きしよう．まずは，simversion.hの\verb|SIM_SAVE_MINOR|の値を増やす．お手元のコードの\verb|SIM_SAVE_MINOR|が7ではなかった場合はお手元の\verb|SIM_SAVE_MINOR|の値を1増やせばよい．

\begin{lstlisting}[caption=simversion.h抜粋（13行めから）, style=customC]
// もともとの値が7だったので1増やして8にする
// SIM_SERVER_MINORも増やす
#define SIM_SAVE_MINOR      8
#define SIM_SERVER_MINOR    8
\end{lstlisting}

つづいて，dataobj/settings.ccの\verb|settings_t::rdwr()|に\verb|max_cityroad_speed|の読み書き処理を追記する．\verb|settings_t|オブジェクトの初期化時点で\verb|max_cityroad_speed|は初期化されているので，バージョン番号が十分新しい場合にのみ読み書き処理をすれば十分である．適当に\verb|settings_t::rdwr()|の末尾にでも追記しておけばよい．

\begin{lstlisting}[caption=max\_cityroad\_speedをセーブデータに保存する, style=customC]
if(  file->get_version() >= 120008  ) {
	file->rdwr_short(max_cityroad_speed);
}
\end{lstlisting}

\subsection{高度な設定ウィンドウの整備}
\verb|max_cityroad_speed|の初期化はした．simuconfのパースもした．パラメータのファイル保存もした．市道化アルゴリズムは\verb|max_cityroad_speed|を参照するようになった．やり残したことは，ユーザーが「高度な設定」ウィンドウから\verb|max_cityroad_speed|を編集できるようにすることである．これをやらないと，ユーザーは一度ゲームを開始したら二度とそのパラメータを変更できないことになってしまう．

\begin{figure}
  \centering
  \includegraphics[width=7cm]{images/市道化防止/4.png}
  \caption{高度な設定 の 経済タブ}
  \label{高度な設定ウィンドウ}
\end{figure}

高度な設定ウィンドウは図\ref{高度な設定ウィンドウ}のように，大きな外枠ウィンドウがあってその中に6つのタブが並んでいる．\verb|max_cityroad_speed|は「経済」タブの中に配置するのが妥当であろう．というわけで，経済タブの一番下に配置することにしよう．

次に，ソースコードフォルダのguiディレクトリを眺めてみると，settings\_frame.h・.ccとsettings\_stats.h・.ccを見つけることができる．settings\_frame.hおよびsettings\_frame.ccは高度な設定ウィンドウの外枠を定義しているファイルである．settings\_stats.hおよびsettings\_stats.ccは各パラメータの設定部品を定義しており，我々が変更を加えるのはこちらである．このファイルはマクロを使って書かれており，コードを文法的にまじめに解読する意味はない．gui/settings\_stats.ccにある，\verb|settings_economy_stats_t::init|と\verb|settings_economy_stats_t::read|の末尾にそれぞれ次のように追記しよう．

\begin{lstlisting}[caption=高度な設定ウィンドウの整備, style=customC, label=settings_stat]
void settings_economy_stats_t::read(settings_t* const sets)
{
  // （この間たくさんの初期化コード）
  READ_NUM_VALUE( sets->max_cityroad_speed );
}

void settings_costs_stats_t::init(settings_t const* const sets) {
  // （この間たくさんの初期化コード）
  // 引数は 表示名, 変数, min, max, 補完, wrap (通常false)
  INIT_NUM("max_cityroad_speed", sets->get_max_cityroad_speed, 0, 65535, gui_numberinput_t::AUTOLINEAR, false );

  clear_dirty();
  height = ypos;
  set_size( settings_stats_t::get_size() );
}
\end{lstlisting}
\verb|settings_t|において\verb|max_cityroad_speed|はprivate宣言したにもかかわらずコード\ref{settings_stat}の10行目では\verb|max_cityroad_speed|が直接アクセスされている．これは，このクラスがfriendクラス扱いになっているからである．friendクラスについては『ロベールのC++教室』第2部第43章（\url{http://www7b.biglobe.ne.jp/~robe/cpphtml/html02/cpp02043.html}）を読むとよい．

これで市道化境界速度のパラメータが正しく設定できるようになり，ファイルに保存され，市道化処理に反映されるようになった．所望の結果が得られているか，各自でテストしてみよう．

\section{改造2: 市道化の可否を道路1タイルずつ設定する}
\label{改造2}

２つめのアプローチは，市道化の可否を道路1タイルずつ指定する方式である．道路建設ツールをctrlキーを押しながらクリックすると，図\ref{OTRP_road_config}のように市道化防止ボタンが登場する．市道化防止ボタンが押された状態でその道路を建設すると，その道路についてのみ市道化が禁止される．

先ほどの改造とは全く別の改造になるので，gitで新しいブランチを用意しよう．コマンドラインで下のようにすれば，「cityroad\_button」という名前のブランチを作成し，同時にそのブランチに切り替えることができる．cityroad\_buttonブランチはspeed\_thresholdブランチからではなく，masterブランチから派生させることに注意されたい．

\begin{lstlisting}[caption=gitでcityroad\_buttonブランチを切る, style=bash]
git checkout master
git checkout -b cityroad_button
\end{lstlisting}

まず，市道化の可否はどうすればいいのだろう．道路オブジェクトに市道化可否フラグを持たせて，\verb|stadt_t::update_city_street()|の中で対象道路に市道化防止フラグが立っているか検査してから市道化処理を行えば良さそうである．道路に変数をもたせたので，それをセーブデータに保存することも必要だ．変数をセーブデータから出し入れするには\verb|rwdr()|関数を使えばよいのであった．あとは，GUIから市道化可否フラグを設定できればよい．フラグは道路建設ツールを使って設定すると良いだろう．したがって，道路建設ツールがユーザーに対して設定ウィンドウを提供する．ユーザーが実際に道路を建設するときに，ツールが一緒に道路オブジェクトにフラグを設定すればよい．

ここまでで基本的な方針はついた．やることはもう分かっているので，あとは編集したいクラスが書かれているファイルを見つけ，編集すべき関数を見つけ，内容を理解し，コードを書き換えればよい．道路を建設するツールは何というクラスなのだろうか．クラスや関数の探し方は既に解説したとおりである．動作主体を考え，その単語で横断検索をかけ，あるいはディレクトリ構造からたどっていく．さあ，コードの海へ漕ぎ出そう．\\
\dotfill

独力での実装の旅を続けていると，このトピックではいくつかの難所に遭遇するはずである．このあとの節では，それを一つ一つ解説していこう．

\subsection{street flagとrdwrの整備}
本改造では，市道化をする時にその道路が市道化可能か判断をする．このため，道路オブジェクトに市道化可否を表すフラグ変数が必要である．まずは，道路オブジェクトに市道化可否のフラグを設け，rdwrするところまで進もう．道路オブジェクトを定義するクラスはboden/wege/strasse.hで定義されている\verb|strasse_t|（strasseはドイツ語で道路の意味である．）である．このクラスに変数\verb|street_flags|を定義し，そのgetter・setter関数を定義する．変数と関数をコード\ref{edit_strasse_h}のように追加する．
\begin{lstlisting}[caption=strasse.hの編集, style=customC, label=edit_strasse_h]
class strasse_t : public weg_t
{
public:
  // enumを定義
  enum { AVOID_CITYROAD = 0x01 };
  
private:
  // bool変数として定義するのではなく8bit整数として定義する
  uint8 street_flags;
  
public:
uint8 get_street_flag() const { return street_flags; }
void set_street_flag(uint8 s) { street_flags = s; }
bool get_avoid_cityroad() const { return street_flags&AVOID_CITYROAD; }
void set_avoid_cityroad(bool s) { s ? street_flags |= AVOID_CITYROAD : street_flags &= ~AVOID_CITYROAD; }

};
\end{lstlisting}
bool変数で市道化の可否を定義してもよかったのだが，あえてuint8変数として定義した．このようにすれば，後でフラグを追加するときにenumとgetter・setterの追加で済む．このテクニックはsimutransでは多数箇所で用いられている．例えば，\verb|roadsign_desc_t|（descriptor/roadsign\_desc.h）を見ると，同じように各フラグがuint8変数1つにまとめられていることがわかる．

つづいて，boden/wege/strasse.ccを改変しよう．strasse.ccでは先ほど\verb|strasse_t|に定義した\verb|street_flags|を初期化し，セーブデータに読み書きできるようにする．セーブデータの読み書きは\verb|rdwr()|関数に書けば良いのであった．\verb|strasse_t|のコンストラクタは引数ナシのものとセーブファイルオブジェクトをとるものがあることに注意しよう．セーブファイルオブジェクトを引数に取るコンストラクタで\verb|rdwr()|が呼ばれていることがわかる．コード\ref{strasse_cc}の5行目および14〜19行目が，boden/wege/strasse.ccにおける追記内容である．

\begin{lstlisting}[caption=street\_flagsの読み書き, style=customC, label=strasse_cc]
strasse_t::strasse_t() : weg_t()
{
  set_gehweg(false);
	set_desc(default_strasse);
  street_flags = 0;
}

void strasse_t::rdwr(loadsave_t *file)
{
	xml_tag_t s( file, "strasse_t" );

	weg_t::rdwr(file);
  
  // street_flagsの読み書き
  if(  file->get_version()>=120007  ) {
    file->rdwr_byte(street_flags);
  } else {
    street_flags = 0;
  }
}
\end{lstlisting}

\verb|street_flags|を設定したので，市道化処理を行う\verb|update_city_street()|も\verb|street_flags|を参照するようにする．simcity.ccにある\verb|update_city_street()|をコード\ref{check_avoid_cityroad}のように変更する．8行目にあるif文の条件式の中に市道化可否フラグの項が追加されたことがわかるだろう．\verb|street_flags|は\verb|strasse_t|で定義しているので，コード\ref{check_avoid_cityroad}の6行目では道路オブジェクトを\verb|weg_t|ではなく\verb|strasse_t|で扱っていることに注意しよう．
\begin{lstlisting}[caption=市道化防止フラグが建っていないか検査（simcity.cc）, style=customC, label=check_avoid_cityroad]
bool update_city_street(koord pos)
{
	const way_desc_t* cr = world()->get_city_road();
	for(  int i=0;  i<8;  i++  ) {
		if(  grund_t *gr = world()->lookup_kartenboden(pos+neighbors[i])  ) {
			if(  strasse_t* const weg = (strasse_t*)(gr->get_weg(road_wt))  ) {
				// Check if any changes are needed.
				if(  (!weg->hat_gehweg()  ||  weg->get_desc() != cr)  &&  !weg->get_avoid_cityroad()  ) {
          // 市道化する
					player_t *sp = weg->get_owner();
（以下省略）
\end{lstlisting}

\subsection{市道化可否を道路建設に反映}
道路オブジェクトに市道化可否フラグを設け，\verb|update_city_street()|でそれを参照するようにした．あとは，ユーザーが市道化の可否をGUIで設定し，それが道路建設時に適切に反映されるようにすればよい．まずは，ユーザーが設定した市道化可否を道路建設に反映するようにしよう．ここでもやることは同じ．クラスを特定し，関数を特定し，内容を理解し，改変を加える．まずは自分でコードを読み，「道路建設がどのようにして行われているのか」を把握することにチャンレジしよう．\\
\dotfill

何度か述べたとおり，Simutransで道路を建設しているのは「道路建設ツール」である．ソースコードフォルダを眺めていると，simtool.hというファイルが見当たる．このヘッダファイルを開いて，眺めてみよう．\verb|tool_remover_t|や\verb|tool_raise_t|といったクラスの宣言が見られる．simtool.hはSimutransにおける種々のツールクラスを定義したファイルである．

この中に道路建設ツールをあらわすクラスがあるはずである．「道路」「建設」であるから，way，weg，build，construct，street，strasseなど思いつく限りの関連ワードでsimtool.hの中身を検索してみよう．すると，\verb|tool_build_way_t|というクラスが見つかった．なお，その後ろのほうに\verb|tool_build_cityroad|や\verb|tool_build_bridge_t|，\verb|tool_build_tunnel_t|といったクラスも見つけることができる．それぞれ市道，橋，トンネルの建設クラスであろう．

\verb|tool_build_way_t|に話を戻そう．クラスの機能・性質を知るために，まずはヘッダファイルを読むのであった．25行ほどであるから一つ一つの関数についてその名前から機能を想像してみよう．例えば，\verb|calc_route()|は道路建設ツールでルート検索をしているのだから，与えられた起点終点座標について建設ルートを計算する関数であろう．\verb|init()|はその名の通りツールオブジェクトの初期化関数であるようだ．\verb|do_work()|は「仕事をする」なので，実際に道路建設をする関数であると想像できる．名前やコメントだけではイマイチわからず，その内容を知りたいときは実装（.cc）ファイルを読んで中身の処理を理解しよう．

ともかく，我々は今「市道化可否を道路建設に反映」したいので，実際に道路建設をする関数\verb|do_work()|の中身が気になる．そこで，simtool.ccにかかれている\verb|tool_build_way_t::do_work()|を読むことにしよう．
\begin{lstlisting}[caption=tool\_build\_way\_t::do\_work(), style=customC, label=do_work]
const char *tool_build_way_t::do_work( player_t *player, const koord3d &start, const koord3d &end )
{
	way_builder_t bauigel(player);
	calc_route( bauigel, start, end );
	if(  bauigel.get_route().get_count()>1  ) {
		welt->mute_sound(true);
		bauigel.build();
		welt->mute_sound(false);
		return NULL;
	}
	return "";
}
\end{lstlisting}

コード\ref{do_work}の3行目で\verb|way_builder_t|型の変数を生成している．その後建設ルートを計算し，ルートが有効であれば3行目で生成した\verb|way_builder_t|型変数の\verb|build()|を呼び出している．ここから，道路建設の実体は\verb|way_builder_t|クラスにあることがわかった．そこで，\verb|way_builder_t|クラスを調査しよう．\verb|way_builder_t|のヘッダファイルはbauer/wegbauer.hである．（bauerはドイツ語でbuilderという意味であるからファイルの場所を推定するのは容易であろう．）

bauer/wegbauer.hを見ると，\verb|way_builder_t|は多数の変数と関数を持った大きなクラスであることがわかる．このままヘッダファイルを眺めていてもナニをすればよくわからないが，とりあえずコード\ref{do_work}の7行目で呼ばれた関数\verb|build()|は発見することができる．我々はこの処理の中身が知りたいのであるから，bauer/wegbauer.ccに書かれた\verb|way_builder_t::build()|を読むことにしよう．この関数を読むとその中ほどにコード\ref{way_builder_build}ような記述がある．

\begin{lstlisting}[caption=way\_builder\_t::build()　抜粋, style=customC, label=way_builder_build]
switch(bautyp&bautyp_mask) {
  case wasser:
  case schiene:
  case schiene_tram: // Dario: Tramway
  case monorail:
  case maglev:
  case narrowgauge:
  case luft:
    DBG_MESSAGE("way_builder_t::build", "schiene");
    build_track();
    break;
  case strasse:
    build_road();
    DBG_MESSAGE("way_builder_t::build", "strasse");
    break;
  case leitung:
    build_powerline();
    break;
  case river:
    build_river();
    break;
  default:
    break;
}
\end{lstlisting}

\verb|way_builder_t|オブジェクトの\verb|bautyp|変数に予めwaytypeが設定されていて，それに応じて呼ばれる関数が変化する．今回は道路建設ツールの改造なので\verb|build_road()|が呼ばれる．どうやらこの関数が道路建設の本体のようである．（ちなみに，\verb|bautyp|はコード\ref{do_work}の4行目\verb|calc_route|で設定されている．気になる人は\verb|calc_route()|も読んでほしい．）

\verb|way_builder_t::build_road()|を読んでみよう．同じくbauer/weg\_bauer.ccに記述されている．80行程度のコードであり紙面に貼ることはしないので，お手元のコードを眺めながら読み進めてほしい．冒頭で市道・undoまわりの処理をした後，予め計算した建設ルート1マスずつに対してfor文で処理を回していく．タイルが高架タイルorトンネルタイルなら処理をスキップする．既存道路をアップグレードするのか新規建設するのかで場合分けをし，それぞれについて道路オブジェクトのdescriptorを設定したりownerを設定したりしている．最後に再描画処理を呼ぶ．

simtool.hの探索から始まって，ようやく道路建設処理の全体が見えた．ここまでの流れを簡単に整理しておこう．\verb|tool_build_way_t|が道路建設ツールであり，その\verb|do_work()|関数が道路建設を行う．\verb|do_work()|は\verb|way_builder_t|に建設処理を丸投げしており，\verb|build()|を経由して\verb|build_road()|が呼ばれる．\verb|build_road()|は予め設定された建設ルートにそって1マスずつ道路オブジェクトを配置し，道路オブジェクトに対して設定をしていく．

処理の全体がわかった今，ユーザーがウィンドウ経由で設定した\verb|street_flags|（\verb|strasse_t|で定義したのをおぼえているだろうか）を道路オブジェクトに反映させたい．\verb|tool_build_way_t|に\verb|street_flags|変数を設けた上で（あとでこれをGUIで編集できるようにする），改めて自分で手を動かしてコード改変にチャレンジしてみよう．\\
\dotfill

まずは，way\_builderから手を付けよう．\verb|build()|や\verb|build_road()|はそれ自体は引数を取らず，予めオブジェクトに設定しておいた値を読んで処理する方式である．そこで，まずは\verb|way_builder_t|クラスに変数\verb|street_flag|を設定する．bauer/wegbauer.hに以下のように追記することで，\verb|street_flag|を定義し，setter関数を設ける．getter関数を定義していないのは，\verb|tool_build_way_t|が\verb|way_builder_t|に建設作業を投げて実際に建設を行う際に\verb|way_builder_t|の\verb|street_flag|を読み出す必要が無いからである．
\begin{lstlisting}[caption=bauer/wegbauer.h　追記, style=customC]
class way_builder_t
{
private:
  uint8 street_flag;
  
public:
  void set_street_flag(uint8 a) { street_flag = a; }
};
\end{lstlisting}

次に\verb|build_road()|でこの\verb|street_flag|を道路オブジェクトに設定していく．道路オブジェクトに対する操作なので，位置関係的にはdescriptorを設定している行の前後あたりに書けばよいだろう．以下に一部省略した\verb|way_builder_t::build_road()|のコードを示す．

\begin{lstlisting}[caption=build\_road()でstreet\_flagの設定, style=customC, label=str_flag1]
void way_builder_t::build_road() {
  // 市道やundoまわりの処理．掲載省略．
	for(  uint32 i=0;  i<get_count();  i++  ) {
    if((i&3)==0) {
			INT_CHECK( "wegbauer 1584" );
		}

		const koord k = route[i].get_2d();
		grund_t* gr = welt->lookup(route[i]);
		sint64 cost = 0;

		bool extend = gr->weg_erweitern(road_wt, route.get_short_ribi(i));

		// 橋やトンネルの場合はstreet_flagだけアップデートする
		if(gr->get_typ()==grund_t::brueckenboden  ||  gr->get_typ()==grund_t::tunnelboden) {
      strasse_t* str = (strasse_t*)gr->get_weg(road_wt);
			str->set_street_flag(street_flag);
      continue;
		}

		if(extend) {
      // 型はweg_tではなくstrasse_tにする
			strasse_t * weg = (strasse_t*)(gr->get_weg(road_wt));

      if(gr->get_typ()==grund_t::monorailboden && (bautyp&elevated_flag)==0) {
        // 高架の場合street_flagだけアップデートする
				weg->set_street_flag(street_flag);
			}
			// keep faster ways or if it is the same way ... (@author prissi)
			else if((weg->get_desc()==desc  &&  weg->get_overtaking_mode()==overtaking_mode  &&  weg->get_street_flag()==street_flag  )  ||  keep_existing_ways  ||  (keep_existing_city_roads  &&  weg->hat_gehweg())  ||  (keep_existing_faster_ways  &&  weg->get_desc()->get_topspeed()>desc->get_topspeed())  ||  (player_builder!=NULL  &&  weg->is_deletable(player_builder)!=NULL)) {
				//nothing to be done
			}
			else {
				// we take ownership => we take care to maintain the roads completely ...
				player_t *s = weg->get_owner();
				player_t::add_maintenance(s, -weg->get_desc()->get_maintenance(), weg->get_desc()->get_finance_waytype());
				// cost is the more expensive one, so downgrading is between removing and new building
				cost -= max( weg->get_desc()->get_price(), desc->get_price() );
        // street_flagの設定
        weg->set_street_flag(street_flag);
				weg->set_desc(desc);
        // 以下省略
			}
		}
		else {
			// make new way
			strasse_t * str = new strasse_t();
			str->set_desc(desc);
      // street_flagの設定
      str->set_street_flag(street_flag);
			str->set_gehweg(add_sidewalk);
			// 以下省略
		}
		// 再描画まわりの処理．掲載省略．
	} // for
}
\end{lstlisting}

コード\ref{str_flag1}ではまず，16行目でトンネルや橋の場合について，26行目で高架の場合についても\verb|street_flag|をアップデートするようにした．この都合でif文の構造が変わっていることに注意されたい．道路置き換えの場合は39行目，新規建設の場合は49行目でそれぞれ\verb|street_flag|を設定している．編集したのはこの4点で，それ以外はもとのコードと同じである．

あとは，\verb|tool_build_way_t|が\verb|way_builder_t|に道路建設をさせるときに\verb|street_flag|を\verb|way_builder_t|オブジェクトに設定すればよい．\verb|tool_build_way_t|に\verb|street_flag|変数を設けた上で（コード\ref{flag_on_simtool_h}），\verb|tool_build_way_t::do_work()|をコード\ref{do_work_mod}のように変更すればよいであろう．コード\ref{do_work_mod}では5行目で\verb|street_flag|の設定を行っている．

\begin{lstlisting}[caption=simtool.hへの追記, style=customC, label=flag_on_simtool_h]
class tool_build_way_t : public two_click_tool_t {
protected:
  uint8 street_flag = 0;
  
pubic:
  void set_street_flag (uint8 a) { street_flag = a; }
  uint8 get_street_flag() const { return street_flag; }
};
\end{lstlisting}

\begin{lstlisting}[caption=tool\_build\_way\_t::do\_work()の変更（simtool.cc）, style=customC, label=do_work_mod]
const char *tool_build_way_t::do_work( player_t *player, const koord3d &start, const koord3d &end )
{
	way_builder_t bauigel(player);
	calc_route( bauigel, start, end );
  bauigel.set_street_flag(street_flag);
	if(  bauigel.get_route().get_count()>1  ) {
		welt->mute_sound(true);
		bauigel.build();
		welt->mute_sound(false);
		return NULL;
	}
	return "";
}
\end{lstlisting}

\subsection{street\_flagをGUIで設定する}
道路オブジェクトに市道化防止フラグは設けた．道路建設ツールが道路オブジェクトにフラグを正しく設定できるようにもした．あとは，ユーザが市道化防止フラグをGUIで設定できるようにすればよい．我々が実現したいのは図\ref{OTRP_road_config}のようなウィンドウである．ctrlキーを押せばウィンドウがポップアップしてきて，ユーザーがその中のボタンを押せばそれが\verb|tool_build_way_t|の\verb|street_flag|変数に反映されればよい．そうすれば，建設時に\verb|tool_build_way_t::do_work()|が\verb|street_flag|を\verb|way_builder_t|に適切に渡してくれる．

それでは，ctrlキーを押しながら道路建設ツールを呼んだときウィンドウをポップアップして市道化防止オプションボタンを表示するにはどうすればいいのだろうか？どうすればいいのかよくわからないときは，{\bf 似たようなことをやっている事例を探してきて，そこのコードをコピーする}のがスムーズな方法である．特に，GUIまわりのコードは内容を詳細に理解しようとするよりもとりあえず動いている既存のコードをコピペして使ってしまうほうが開発がスムーズに進行する．では，「ctrlキーを押しながらツールを呼ぶとウィンドウが出てきてオプションボタンを押せる」ことと似たようなことをやっているのは何だろう？

\begin{figure}
  \centering
  \includegraphics[width=5cm]{images/市道化防止/5.png}
  \caption{信号・標識の間隔設定ウィンドウ}
  \label{railway_sign_img}
\end{figure}

それは，図\ref{railway_sign_img}のような信号・標識の間隔設定ウィンドウである．したがって，信号・標識がどのようにしてこのような機能を実現しているのかを理解すればよい．まずは，simtool.hから信号設置ツールのクラスを見つけてみよう．signal, signといった関連ワードで検索をかけてみると\verb|tool_build_roadsign_t|というクラスが見つかる．これが信号・標識を設置するツールである．ヘッダファイルで\verb|tool_build_roadsign_t|を眺めてもあまりかわりばえしないので，実装ファイルを覗く必要がある．とりあえず，初期化関数っぽい\verb|tool_build_roadsign_t::init()|をのぞいてみるとしよう．コード\ref{roadsign_init}がそれである．

\begin{lstlisting}[caption=tool\_build\_roadsign\_t::init()　in simtool.cc, style=customC, label=roadsign_init]
bool tool_build_roadsign_t::init( player_t *player)
{
	desc = roadsign_t::find_desc(default_param);
	// take default values from players settings
	current = signal[player->get_player_nr()];

	if (is_ctrl_pressed()  &&  can_use_gui()) {
		create_win(new signal_spacing_frame_t(player, this), w_info, (ptrdiff_t)this);
	}
	return two_click_tool_t::init(player)  &&  (desc!=NULL);
}
\end{lstlisting}
7〜9行目に注目してほしい．ctrlキーが押されていたらウィンドウを作れと書いてある．そのウィンドウは\verb|signal_spacing_frame_t|で定義されている．ctrlキーを押しながらツールをクリックしたらウィンドウが出てくるようにするには，ウィンドウクラスを定義した上でコード\ref{roadsign_init}の7〜9行目のようなコードを書けばよいことがわかった．

では，\verb|signal_spacing_frame_t|を調査しよう．このクラスはgui/signal\_spacing.hに書かれている．まずは定石どおりにヘッダファイルを見てほしい．private修飾子の中には信号間隔や設置オプションといったこのウィンドウで編集するパラメータ，player，呼び出し元ツール，numberinputやlabel，buttonといったguiコンポーネントの宣言が並ぶ．GUI系クラスのヘッダファイルで用意すべき変数は大きく分けて「制御するパラメータ」，「player・呼び出し元ツールといった決まった変数」，そして「ウィンドウで使うguiコンポーネント」の3つである．public修飾子の中にはコンストラクタ，\verb|action_triggered()|，ヘルプファイル名を返す関数が並ぶ．

それでは，gui/signal\_spacing.ccを調査しよう．お手元で当該ファイルを開きながら以下を読み進めてほしい．ここできちんと記述する必要がある関数はコンストラクタと\verb|action_triggered()|の2つである．まずは\verb|action_triggered()|の方から見てみよう．この関数は登録したguiコンポーネントに何らかのイベントが発生したとき呼ばれる関数である．\verb|action_triggered()|には引数としてイベントを引き起こしたguiコンポーネント\verb|komp|と，そのイベントの値が渡されてくる．今回は，値の方は使っていないので\verb|komp|に注目すればよい．ウィンドウ内のguiコンポーネントと\verb|komp|と比較し，一致すればそのコンポーネントに対して処理を行う．例えば，\verb|komp == &remove_button|であれば，removeの状態を反転させ，ボタンに反映させている．最後に呼び出し元ツールに変更した値を代入している（\verb|return true;|の1行前）．Simutransで提供されるbuttonオブジェクトはボタンを押しても勝手に状態が反転するわけではないことに注意されたい．

つづいて，コンストラクタについて調査しよう．渡されたplayer, 呼び出し元ツールなどを整理した後，各guiコンポーネントの配置処理を行っていることがわかるだろう．例えば，押しボタンの配置処理ならコード\ref{button_process}が配置処理のひとかたまりとなる．また，コンポーネント配置座標として\verb|scr_coord cursor|がsignal\_spacing.ccの29行目で定義されている．

\begin{lstlisting}[caption=buttonを配置, style=customC, label=button_process]
remove_button.init( button_t::square_state, "remove interm. signals", cursor );
remove_button.set_width( L_DIALOG_WIDTH - D_MARGINS_X );
remove_button.add_listener(this);
remove_button.pressed = remove;
add_component( &remove_button );
cursor.y += remove_button.get_size().h + D_V_SPACE;
\end{lstlisting}

コード\ref{button_process}では1行目で\verb|remove_button|を初期化（引数としてボタンの種類，ボタン横のラベル文字列，位置を取る）している．2行目で幅を決めて，3行目listenerに登録（これによって\verb|action_triggered()|が呼ばれるようになる）する．4行目で押されたか状態を設定し，5行目でウィンドウにguiコンポーネントを登録する．最後の6行目で配置座標のy座標をズラしている．これでボタンの配置が完了する．

以上で，信号・標識設置ツールがどのようにしてGUIによる値設定を可能にしているのかが理解できた．toolの\verb|init()|でctrlキーが押されていたらウィンドウを生成し，ウィンドウ内ではguiコンポーネントを並べ，適切に初期化し，ボタンが押されたらそれに対して反応してユーザーが設定した値をtoolに格納する．このやりかたをそっくりそのまま\verb|tool_build_way_t|でもやればいいのである．

なお，設置ツール初期化時に必要に応じてウィンドウを生成するだけでは，設置ツールが使われなくなったときに自動でウィンドウが消えてくれない．\verb|tool_build_roadsign_t|には\verb|exit()|という関数が実装されており（コード\ref{exit_function}），そこでウィンドウの破壊処理が行われている．\verb|init()|や\verb|exit()|の呼び出しはこの手のツール共通の親クラスである\verb|two_click_tool_t|で書かれているので，我々は\verb|exit()|の中にウィンドウ破壊処理を書くだけでよい．忘れずに\verb|tool_build_way_t|にも移植しよう．

\begin{lstlisting}[caption=ウィンドウ破壊処理はexit()に書く, style=customC, label=exit_function]
bool tool_build_roadsign_t::exit( player_t *player )
{
	destroy_win((ptrdiff_t)this);
	return two_click_tool_t::exit(player);
}
\end{lstlisting}

例によって以下の解説に入る前に，まずは自力でGUIダイアログの実装にチャレンジしてほしい．\\
\dotfill

まずは，GUI画面を記述するファイルを作ろう．先ほどの信号・標識設置の場合ではsignal\_spacing.h・.ccに相当する．ここではファイル名はroad\_config.h・.ccとする．ヘッダファイル（gui/road\_config.h）はgui/signal\_spacing.hを参考にすると以下のようになる．今回置くguiコンポーネントは市道化防止ボタンただ一つだけである．

\begin{lstlisting}[caption=gui/road\_config.h, style=customC]
#ifndef road_config_h
#define road_config_h

#include "gui_frame.h"
#include "components/action_listener.h"

class button_t;
class tool_build_way_t;
class player_t;

class road_config_frame_t : public gui_frame_t, private action_listener_t
{
private:
	static uint8 street_flag;
	player_t *player;
	tool_build_way_t* tool;
	button_t button;

public:
	road_config_frame_t( player_t *, tool_build_way_t * );
	bool action_triggered(gui_action_creator_t*, value_t) OVERRIDE;
	const char * get_help_filename() const { return "road_config.txt"; }
};

#endif
\end{lstlisting}

つづいて，gui/signal\_spacing.ccを参考にしてgui/road\_config.ccを書く．実装する関数はコンストラクタと\verb|action_triggered()|の2つである．

\begin{lstlisting}[caption=gui/road\_config.cc, style=customC, label=road_conf_cc]
#include "components/gui_button.h"

#include "road_config.h"
#include "../simtool.h"
#include "../boden/wege/strasse.h"

#define L_DIALOG_WIDTH (200)

uint8 road_config_frame_t::street_flag = 0;

road_config_frame_t::road_config_frame_t(player_t *player_, tool_build_way_t* tool_) :
	gui_frame_t( translator::translate("configure road") )
{
	player = player_;
	tool = tool_;
	street_flag = tool->get_street_flag();

	scr_coord cursor(D_MARGIN_LEFT, D_MARGIN_TOP);

	button.init( button_t::square_state, "avoid becoming cityroad", cursor );
	button.set_width( L_DIALOG_WIDTH - D_MARGINS_X );
	button.add_listener(this);
	button.pressed = weg->set_street_flag(street_flag);;
	add_component( &button );
	cursor.y += button.get_size().h + D_V_SPACE;

	set_windowsize( scr_size( L_DIALOG_WIDTH, D_TITLEBAR_HEIGHT + cursor.y + D_MARGIN_BOTTOM ) );
}

bool road_config_frame_t::action_triggered( gui_action_creator_t *komp, value_t)
{
	if( komp == &button ) {
    button.pressed = !button.pressed;
    if(  button.pressed  ) {
      // AVOID_CITYROADをONにする
      street_flag |= strasse_t::AVOID_CITYROAD;
    } else {
      // AVOID_CITYROADをOFFにする
      street_flag &= ~(strasse_t::AVOID_CITYROAD);
    }
    tool->set_street_flag(street_flag);
	}
	return true;
}
\end{lstlisting}

コード\ref{road_conf_cc}では\verb|strasse_t|のenumを使うため5行目でstrasse.hをincludeしている．9行目で変数の初期化を行い，20〜25行目でbuttonの配置をしている．30行目から\verb|action_triggered|の記述が始まり，ボタンの状態を反転した上で\verb|street_flag|のビット演算をしている．41行目でそれを呼び出し元ツールに戻している．

\verb|road_config_frame_t|クラスが書き上がったので，\verb|tool_build_way_t|から呼び出してあげよう．ウィンドウを閉じるために必要な\verb|exit()|関数は\verb|tool_build_way_t|に実装されていない（オーバーライドされていない）ので，信号・標識と同じようにpublic属性でヘッダファイル（simtool.h）で宣言する．（コード\ref{exit_func_declare}）
\begin{lstlisting}[caption=exit()の宣言, style=customC, label=exit_func_declare]
bool exit(player_t*) OVERRIDE;
\end{lstlisting}

simcity.ccはコード\ref{simcity_mod}のように改変すればよい．road\_config.hを新しく作り，それを使うので1行目でincludeしている．\verb|tool_build_way_t::init()|の中にウィンドウ作成処理を記した．\verb|exit()|関数も新しく\verb|tool_build_way_t|に実装（15行目以降）した．
\begin{lstlisting}[caption=simtool.cc追記, style=customC, label=simcity_mod]
#include "gui/road_config.h"

bool tool_build_way_t::init( player_t *player )
{
	two_click_tool_t::init( player );
  if (is_ctrl_pressed()  &&  can_use_gui()) {
    create_win(new signal_spacing_frame_t(player, this), w_info, (ptrdiff_t)this);
  }
  
  if( ok_sound == NO_SOUND ) {
    ok_sound = SFX_CASH;
  }
（以下省略）

bool tool_build_way_t::exit( player_t *player )
{
	destroy_win((ptrdiff_t)this);
	return two_click_tool_t::exit(player);
}
\end{lstlisting}

これでウィンドウから市道化可否をON/OFFできるようになり，我々の目的は達成された，はずである．最後に，simversion.hの\verb|SIM_SAVE_MINOR|を忘れずに1増やしておこう．（\verb|strasse_t|の\verb|rdwr()|で使ったバージョン番号と合っているか確認する．詳しくは第\ref{セーブデータの読み書き}節を参照されたい．）

ところが，嬉々としてコンパイルを実行すると残念ながら次のエラーに出くわすであろう．

\begin{lstlisting}[caption=最後のLDで失敗するコンパイラの出力, style=bash]
===> LD  /Users/XXXX/sim
Undefined symbols for architecture x86_64:
  "road_config_frame_t::road_config_frame_t(player_t*, tool_build_way_t*, bool)", referenced from:
      tool_build_way_t::init(player_t*, bool) in simtool.o
ld: symbol(s) not found for architecture x86_64
clang: error: linker command failed with exit code 1 (use -v to see invocation)
make: *** [/Users/XXXX/sim] Error 1
\end{lstlisting}

symbols not found，つまり，コンパイラが「road\_config\_frame\_tとか知りません」と怒っている．これはさきほど作成したroad\_config.ccがMakefileに登録されておらず，このファイルに対するコンパイルが行われていないからである．よって，road\_config.ccをコンパイルするようMakefileに追記しなければならない．

simutransソースコードディレクトリのトップ層にMakefileというファイルがあるので，それを開くとおよそ200〜500行目にわたってコード\ref{コンパイル対象ファイルの追加}
\begin{lstlisting}[caption=コンパイル対象ファイルの追加, style=customC, label=コンパイル対象ファイルの追加]
SOURCES += gui/convoi_filter_frame.cc
SOURCES += gui/convoi_frame.cc
SOURCES += gui/convoi_info_t.cc
SOURCES += gui/convoy_item.cc
\end{lstlisting}
のように，ccファイルたちがコンパイル対象として追加されている．よって，road\_config.ccをコンパイル対象として追加するためにはMakefileの適当な場所に
\begin{lstlisting}[caption=Makefileにroad\_config.ccを登録, style=customC]
SOURCES += gui/road_config.cc
\end{lstlisting}
と追記すればよい．これでコンパイルが通ったはずだ．起動してctrlキーを押しながら道路建設アイコンをクリックすれば，図\ref{simple_conf_window}のようなウィンドウで市道化をコントロールできるはずである．

\begin{figure}
  \centering
  \includegraphics[width=5cm]{images/市道化防止/7.png}
  \caption{完成した市道化可否コントロールウィンドウ}
  \label{simple_conf_window}
\end{figure}

\subsection{ネットワークゲーム対応}
先の節でこの改造は一応の完成を見た，ということになった．しかし，実はこのままの状態で世の中にリリースするとユーザーから不具合報告が来ることになる．現時点で残っている不具合は以下の２つである．
\begin{itemize}
  \item 市道化防止が原理的に不要なはずの高架で市道化可否選択ダイアログが表示される．
  \item ネットワークゲームでこの機能を使うと市道化防止が有効にならない．
\end{itemize}
1つめの問題はさほど深刻ではないので放置するとして，ネットワークゲームで市道化防止が機能しないのは大きな問題である．しかし，どうしてこのようなことになってしまったのだろうか．バグを調査して，修正しよう．今回の不具合はネットワークゲームでの不具合である．ネットワークゲームのデバッグをするにはローカルでサーバとクライアントを建てればよい．サーバーは\verb|-server|オプションをつけてsimutransを起動し，クライアントは接続先に\verb|127.0.0.1|（ローカル・ループバック・アドレス）を指定すればよい．

そもそも\verb|tool_build_way_t::do_work()|で\verb|street_flag|は正しく設定されているのだろうか．それを調べるところから始めよう．\verb|tool_build_way_t::do_work()|の5行目（\verb|street_flag|を設定している行．コード\ref{tool_break_point}を参照のこと）にブレークポイントを仕掛ける．（例：その行がsimtool.ccの2438行目であれば，\verb|b simtool.cc:2438| とgdbに入力）
\begin{lstlisting}[caption=この5行目にブレークポイントを設定（simtool.cc）, style=customC, label=tool_break_point]
const char *tool_build_way_t::do_work( player_t *player, const koord3d &start, const koord3d &end )
{
	way_builder_t bauigel(player);
	calc_route( bauigel, start, end );
	bauigel.set_street_flag(street_flag);
  if(  bauigel.get_route().get_count()>1  ) {
		welt->mute_sound(true);
		bauigel.build();
\end{lstlisting}

ブレークポイントを設定したら「\verb|r -server|」でsimutransを起動．市道化防止を有効にして道路を建設してみよう．道路を建設するとブレークポイントに当たってsimutransがフリーズし，gdbが入力を受け付けるようになる．ここで\verb|street_flag|の値を出力してみよう．コード\ref{市道化防止を有効にしたのに}のようにデバッガで\verb|street_flag|を出力させると，市道化防止を有効にしたはずなのになんと\verb|street_flag|は0だと表示された．

\begin{lstlisting}[caption=市道化防止を有効にしたのにstreet\_flagは0, style=bash, label=市道化防止を有効にしたのに]
(gdb) p street_flag
(uint8) $0 = '\0'
\end{lstlisting}

なるほど．では，ctrlキーを押して出てくるあのウィンドウから\verb|street_flag|がきちんと設定されていなかったのではないか．もしくは外部から\verb|street_flag|が0に書き換えられたのではないか．そこで，\verb|tool_build_way_t::set_street_flag()|にコード\ref{observe_street_flag}の3行目のようにprintf文を仕込む．これで，外部から\verb|street_flag|が操作されれば，それはprintf文によって出力されることになる．
\begin{lstlisting}[caption=street\_flagの値変化を観察する（simtool.cc） , style=customC, label=observe_street_flag]
class tool_build_way_t : public two_click_tool_t {
  void set_street_flag(uint8 a) {
		printf("flag:%d\n", a);
		street_flag = a;
	}
};
\end{lstlisting}
ところが，これをコンパイルしネットワークモードで実行し，ウィンドウから市道化防止フラグをONにして道路を建設してもコンソール画面にはコード\ref{不審な操作は見られない}のように出力される．すなはち，\verb|tool_build_way_t|の\verb|street_flag|はただ一度「1」に変更されただけという結果である．
\begin{lstlisting}[caption=street\_flagに不審な操作は見られない, style=bash, label=不審な操作は見られない]
flag:1
\end{lstlisting}

\verb|tool_build_way_t|の\verb|street_flag|はたしかに設定ウィンドウによって正しく1になった．しかし，\verb|do_work()|の段階では0になってしまう．外部からの値操作がないとすると，\verb|tool_build_way_t|で値が書き換えられたのだろうか？そうアタリをつけて原因箇所を探ってみても，残念ながら手がかりは得られない．ここで発想の転換が必要である．市道化防止設定ウィンドウを扱っている\verb|tool_build_way_t|オブジェクトと，\verb|do_work()|を実行している\verb|tool_build_way_t|オブジェクトは，実は別モノなのではないかと．

そこで，それぞれのオブジェクトのメモリ上のアドレスを見てあげることにしよう．これもprintfで見てあげればよい．\verb|set_street_flag()|において先ほど設定値をprintf出力したが，オブジェクトアドレスも追加で出力する．（コード\ref{オブジェクトのアドレス表示}）

\begin{lstlisting}[caption=street\_flagが編集されたオブジェクトのアドレス表示（simtool.cc） , style=customC, label=オブジェクトのアドレス表示]
class tool_build_way_t : public two_click_tool_t {
  void set_street_flag(uint8 a) {
		printf("flag:%d, addr:%d\n", a, this);
		street_flag = a;
	}
};
\end{lstlisting}

同時に\verb|do_work()|を実行しているオブジェクトのアドレスも出力する．
\begin{lstlisting}[caption=do\_work()実行オブジェクトのアドレス表示 , style=customC]
const char *tool_build_way_t::do_work( player_t *player, const koord3d &start, const koord3d &end )
{
	printf("do_work: addr:%d\n", this);
	way_builder_t bauigel(player);
	calc_route( bauigel, start, end );
（以下省略）
\end{lstlisting}

これをコンパイルし，ネットワークモードで起動して市道化防止を設定し道路建設を行うと以下の結果を得る．
\begin{lstlisting}[caption=2つのオブジェクトのアドレスの値は異なっていた, style=bash]
flag:1, addr:11225504
do_work: addr:719768224
\end{lstlisting}

２つのアドレスは異なっている．すなはち，市道化防止設定ウィンドウ経由で\verb|street_flag|を編集したオブジェクトと，\verb|do_work|を実行しているオブジェクトは別モノなのである．だから，\verb|do_work|を実行すると\verb|street_flag|の変更は反映されなかったのだ．

ちなみに，ローカルゲームの場合はこの2つのアドレス出力は一致する．オブジェクトが一致しているので\verb|street_flag|は正しく設定されていたのである．実は，ネットワークゲームで道路建設をするとき，クライアントは道路建設コマンドを発行しているだけである．サーバーはクライアントの発行したコマンドをネットワークごしに受け取り，\verb|do_work()|を実行したらその結果を全てのクライアントに配信しているのである．市道化防止設定ウィンドウを扱っていた\verb|tool_build_way_t|オブジェクトはクライアント側のオブジェクト，\verb|do_work()|を実行したオブジェクトはサーバー側のオブジェクトなのである．一般に，ネットワークゲームではユーザーの相手をするツールオブジェクトと実際に動作を行うツールオブジェクトは別モノである．

これではツールが何かしらのパラメータを使って作業を行おうとしても，それを実行するオブジェクトに情報が伝わらないので困ってしまう．そこで，simutransでは\verb|rdwr_custom_data()|という関数が用意されている．これは親クラス\verb|tool_t|が提供している関数であり，セーブデータ読み書きの\verb|rdwr()|のようにこの中でネットワーク越しにパラメータを読み書きする関数である．\verb|rdwr_custom_data()|は\verb|tool_build_way_t|ではオーバーライドされていないが，\verb|tool_build_bridge_t|ではオーバーライドされているのでこれを真似て実装しよう．

ヘッダファイルで\verb|rdwr_custom_data()|を宣言し，実装ファイルで実装してあげればよい．それぞれコード\ref{rdwr_custom_data_h}と\ref{rdwr_custom_data_cc}のように追記する．これでめでたくネットワークゲームでも機能するようになった．
\begin{lstlisting}[caption=simtool.h, style=customC, label=rdwr_custom_data_h]
class tool_build_way_t : public two_click_tool_t {
  void rdwr_custom_data(memory_rw_t*) OVERRIDE;
};
\end{lstlisting}

\begin{lstlisting}[caption=simtool.cc, style=customC, label=rdwr_custom_data_cc]
void tool_build_way_t::rdwr_custom_data(memory_rw_t *packet)
{
	two_click_tool_t::rdwr_custom_data(packet);
	uint8 i = street_flag;
	packet->rdwr_byte(i);
	street_flag = i;
}
\end{lstlisting}

tool系の開発をするときはこのようにネットワークゲームでも所望の動作をするか細心の注意を払って開発する必要がある．
