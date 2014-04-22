# Pack-файлы

Вернёмся к базе объектов в нашем тестовом репозитории. К этому моменту их должно быть 11 штук: 4 блоба, 3 дерева, 3 коммита и одна метка:

	$ find .git/objects -type f
	.git/objects/01/55eb4229851634a0f03eb265b69f5a2d56f341 # tree 2
	.git/objects/1a/410efbd13591db07496601ebc7a059dd55cfe9 # commit 3
	.git/objects/1f/7a7a472abf3dd9643fd615f6da379c4acb3e3a # test.txt v2
	.git/objects/3c/4e9cd789d88d8d89c1073707c3585e41b0e614 # tree 3
	.git/objects/83/baae61804e65cc73a7201a7252750c76066a30 # test.txt v1
	.git/objects/95/85191f37f7b0fb9444f35a9bf50de191beadc2 # tag
	.git/objects/ca/c0cab538b970a37ea1e769cbbde608743bc96d # commit 2
	.git/objects/d6/70460b4b4aece5915caf5c68d12f560a9fe3e4 # 'test content'
	.git/objects/d8/329fc1cc938780ffdd9f94e0d364e0ea74f579 # tree 1
	.git/objects/fa/49b077972391ad58037050f2a75f74e3671e92 # new.txt
	.git/objects/fd/f4fc3344e67ab068f836878b6c4951e3b15f3d # commit 1

Git сжал содержимое этих файлов при помощи zlib, к тому же мы не записывали много данных, поэтому все эти файлы вместе занимают всего 925 байт. Для того чтобы продемонстрировать одну интересную возможность Git'а, добавим файл побольше. Добавим файл repo.rb из библиотеки Grit, с которой мы работали ранее, он занимает примерно 12 Кбайт:

	$ curl https://raw.github.com/mojombo/grit/master/lib/grit/repo.rb > repo.rb
	$ git add repo.rb 
	$ git commit -m 'added repo.rb'
	[master 484a592] added repo.rb
	 3 files changed, 459 insertions(+), 2 deletions(-)
	 delete mode 100644 bak/test.txt
	 create mode 100644 repo.rb
	 rewrite test.txt (100%)

Если мы посмотрим на полученное дерево, мы увидим значение SHA-1, которое получил блоб для файла repo.rb:

	$ git cat-file -p master^{tree}
	100644 blob fa49b077972391ad58037050f2a75f74e3671e92      new.txt
	100644 blob 9bc1dc421dcd51b4ac296e3e5b6e2a99cf44391e      repo.rb
	100644 blob e3f094f522629ae358806b17daf78246c27c007b      test.txt

Посмотрим, сколько этот объект занимает места на диске:

	$ du -b .git/objects/9b/c1dc421dcd51b4ac296e3e5b6e2a99cf44391e
	4102	.git/objects/9b/c1dc421dcd51b4ac296e3e5b6e2a99cf44391e

Теперь изменим немного данный файл и посмотрим на результат:

	$ echo '# testing' >> repo.rb 
	$ git commit -am 'modified repo a bit'
	[master ab1afef] modified repo a bit
	 1 files changed, 1 insertions(+), 0 deletions(-)

Взглянув на дерево, полученное в результате коммита, мы увидим любопытную вещь:

	$ git cat-file -p master^{tree}
	100644 blob fa49b077972391ad58037050f2a75f74e3671e92      new.txt
	100644 blob 05408d195263d853f09dca71d55116663690c27c      repo.rb
	100644 blob e3f094f522629ae358806b17daf78246c27c007b      test.txt

Теперь файлу repo.rb соответствует другой объект-блоб. Это означает, что даже одна единственная строка, добавленная в конец 400-строчного файла, требует создания абсолютно нового объекта:

	$ du -b .git/objects/05/408d195263d853f09dca71d55116663690c27c
	4109	.git/objects/05/408d195263d853f09dca71d55116663690c27c

Итак, мы имеем два почти одинаковых объекта занимающих по 4 Кбайта на диске. Было бы неплохо, если бы Git сохранял только один объект целиком, а другой как разницу между ним и первым объектом.

Оказывается, что Git так и делает. Первоначальный формат для сохранения объектов в Git'е называется рыхлым форматом (loose format) объектов. Однако, время от времени Git упаковывает несколько таких объектов в один pack-файл (pack в пер. с англ. — упаковывать, уплотнять) для сохранения места на диске и повышения эффективности. Это происходит, когда "рыхлых" объектов становится слишком много, а также при вызове `git gc` вручную, и при отправке изменений на удалённый сервер. Чтобы посмотреть, как происходит упаковка, можно выполнить команду `git gc`:

	$ git gc
	Counting objects: 17, done.
	Delta compression using 2 threads.
	Compressing objects: 100% (13/13), done.
	Writing objects: 100% (17/17), done.
	Total 17 (delta 1), reused 10 (delta 0)

Если вы загляните в каталог с объектами, вы обнаружите, что большая часть объектов исчезла, зато появились два новых файла:

	$ find .git/objects -type f
	.git/objects/71/08f7ecb345ee9d0084193f147cdad4d2998293
	.git/objects/d6/70460b4b4aece5915caf5c68d12f560a9fe3e4
	.git/objects/info/packs
	.git/objects/pack/pack-7a16e4488ae40c7d2bc56ea2bd43e25212a66c45.idx
	.git/objects/pack/pack-7a16e4488ae40c7d2bc56ea2bd43e25212a66c45.pack

Оставшиеся объекты — блобы, на которые не указывает ни один коммит. В нашем случае это созданные ранее объекты: содержащий строку "есть проблемы, шеф?", и блоб содержащий "test content". В силу того, что ни в одном коммите данные файлы не присутствуют, они считаются "висячими" и не упаковываются.

Остальные файлы — это pack-файл и его индекс. Pack-файл — это файл, который теперь содержит все объекты, которые были удалены. А индекс — это файл, в котором записаны их смещения в pack-файле, что даёт возможность быстро найти нужный объект. Упаковка данных положительно повлияла на общий размер файлов, если до вызова `gc` они занимали примерно 8 Кбайт, то pack-файл занимает всего 4 Кбайт. Упаковкой объектов мы смогли сократить место, занятое на диске, в два раза.

Как Git это делает? При упаковке Git ищет файлы, которые похожи по имени и размеру, и сохраняет только разницу между двумя версиями файла. Можно рассмотреть pack-файл подробнее и понять, какие действия были выполнены для сжатия. Для просмотра содержимого упакованного файла существует служебная команда `git verify-pack`:

	$ git verify-pack -v \
	  .git/objects/pack/pack-7a16e4488ae40c7d2bc56ea2bd43e25212a66c45.idx
	0155eb4229851634a0f03eb265b69f5a2d56f341 tree   71 76 5400
	05408d195263d853f09dca71d55116663690c27c blob   12908 3478 874
	09f01cea547666f58d6a8d809583841a7c6f0130 tree   106 107 5086
	1a410efbd13591db07496601ebc7a059dd55cfe9 commit 225 151 322
	1f7a7a472abf3dd9643fd615f6da379c4acb3e3a blob   10 19 5381
	3c4e9cd789d88d8d89c1073707c3585e41b0e614 tree   101 105 5211
	484a59275031909e19aadb7c92262719cfcdf19a commit 226 153 169
	83baae61804e65cc73a7201a7252750c76066a30 blob   10 19 5362
	9585191f37f7b0fb9444f35a9bf50de191beadc2 tag    136 127 5476
	9bc1dc421dcd51b4ac296e3e5b6e2a99cf44391e blob   7 18 5193 1 \
	  05408d195263d853f09dca71d55116663690c27c
	ab1afef80fac8e34258ff41fc1b867c702daa24b commit 232 157 12
	cac0cab538b970a37ea1e769cbbde608743bc96d commit 226 154 473
	d8329fc1cc938780ffdd9f94e0d364e0ea74f579 tree   36 46 5316
	e3f094f522629ae358806b17daf78246c27c007b blob   1486 734 4352
	f8f51d7d8a1760462eca26eebafde32087499533 tree   106 107 749
	fa49b077972391ad58037050f2a75f74e3671e92 blob   9 18 856
	fdf4fc3344e67ab068f836878b6c4951e3b15f3d commit 177 122 627
	chain length = 1: 1 object
	pack-7a16e4488ae40c7d2bc56ea2bd43e25212a66c45.pack: ok

Здесь блоб `9bc1d`, который, как мы помним, был первой версией файла repo.rb, ссылается на блоб `05408`, который был второй его версией. Третья колонка в выводе — это размер содержимого объекта. Как видите, содержимое `05408` занимает 12 Кбайт, при этом содержимое `9bc1d` занимает всего лишь 7 байт. Что интересно, вторая версия сохраняется "как есть", а исходная — в виде дельты. Это из-за того, что необходимость получения доступа к последней версии файла является более вероятной.

Также здорово, что переупаковку можно выполнять в любое время. Время от времени Git будет выполнять её автоматически, чтобы сэкономить место на диске. Если вдруг этого недостаточно, всегда можно выполнить `git gc` вручную.