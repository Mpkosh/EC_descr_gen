# Генерация описаний товаров на основе фотографии
[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/drive/1RTGB1etgSAcoOScLUOmByagtjPGwNKlW?usp=sharing)

TL;DR: BLEU-4 – 0.066, BERTScore – 0.133. (В статье BLEU-4 – 0.068) 

## Содержание
1. [Описание задачи](#описание-задачи)
2. [Данные](#данные)
3. [Предобработка](#предобработка)

&emsp; &emsp; 3.1 [Изображения товаров](#изображения-товаров)

&emsp; &emsp; 3.2 [Описания товаров](#описания-товаров)

4. [Архитектура модели](#архитектура-модели)

&emsp; &emsp; 4.1 [Encoder ResNet](#encoder-resnet)

&emsp; &emsp; 4.2 [Decoder LSTM](#decoder-lstm)

5. [Обучение](#обучение)
6. [Оценка](#оценка)
7. [Генерация описаний](#генерация-описаний)

&emsp; &emsp; 7.1 [Изображение из тестовой выборки](#изображение-из-тестовой-выборки)

&emsp; &emsp; 7.2 [Изображение из интернета](#изображение-из-интернета)

## Описание задачи
Так как перед получением товар, в отличие от оффлайн магазина, нельзя потрогать и померить, то  для  повышения  товарооборота  продавцы  должны  предоставлять потенциальным покупателям качественную информацию о продаваемой вещи, чтобы  была  возможность проинформировать  покупателя  обо  всех  аспектах товара. К тому же, в обычном магазине присутствуют консультанты, которые могут убедить человека приобрести вещь, а в интернет магазинах мотивировать посетителей на покупку можно только с помощью интересного и цепляющего описания. Также уходит большое количество времени на придумывание нужного описания для каждого товара. 

**Проблема:** уходит много времени для составления описаний, которые являются одним из мотивирующих факторов для покупки в онлайн-магазинах.

**Решение:** генерировать описания на основе изображения.

## Данные
В качестве данных был взят необработанный датасет с описанием товаров из сферы моды (одежда, обувь, аксессуары), который выложен в общий доступ авторами статьи [Fashion Captioning: Towards Generating Accurate Descriptions with Semantic Rewards](https://doi.org/10.48550/arXiv.2008.02693). В датасете представлено 126 753 записей.


## Предобработка
### Изображения товаров
Для задачи генерации описания по фотографии товара было оставлено 18 000 записей, проведено разделение на обучающую и тестовую выборки (15 000 и 3 000 записей соответственно). Предварительно были удалены записи с дублирующимися описаниями.

Все изображения, полученные по ссылкам из датасета, были записаны в два hdf5 файла (по одному на каждую выборку). Все фотографии приведены к размеру 256 на 256 пикселей. Дополнительно был изменен порядок измерений картинок: кол-во каналов, ширина, высота, тк в PyTorch принят такой порядок.

В алгоритме будет задействована предобученная модель ResNet-101, поэтому изображения нужно обработать в соответствии с тем, как модель была обучена: приведение значений пикселей в интервал (0, 1) и их нормализация с помощью приведенных  на сайте значений. Так как 15 000 изображений не такая большая цифра, то дополнительно к изображениям будут применяться трансформации: случайный горизонтальный поворот и случайный наклон от -10 до 10 градусов. 

<p align="center" width="100%">
 <img src="https://github.com/Mpkosh/EC_descr_gen/blob/main/imgs/img_batch.png" width="70%" > 
<p align="center"><i> Фотографии товаров после нормализации и трансформаций</i></p>
</p>  

### Описания товаров
Разбили все описания на токены. Заменили все слова на индексы с помощью словаря, который составлен по обучающей выборке. Словарь отдельно сохранен в json файл с индексами для токенов \<start>, \<end>, \<pad>, \<unk>.

Для того чтобы декодер (LSTM) смог правильно генерировать описания, дополнительно обработаем описания:
1. в начало описания ставится индекс токена \<start>, чтобы модель начала предсказывать,
2. в конец – индекс токена <end>, чтобы модель научилась предсказывать конец описания,
3. доводим все описания до одной длины с помощью индекса токена \<pad>, так как описания будут передаваться как тензоры фиксированной длины,
4. посчитаем длину каждого описания (с токенами начала и конца), чтобы в дальнейшем не делать вычисления с токеном \<pad>.

## Архитектура модели
Реализована архитектура модели из статьи [Show, Attend and Tell: Neural Image Caption Generation with Visual Attention]( 	
https://doi.org/10.48550/arXiv.1502.03044): encoder и decoder с модулем внимания.

### Encoder ResNet
В качестве сверточной сети используется предобученная модель ResNet-101. Так как для модуля внимания нужны признаки в двумерном пространстве и не нужна классификация, то убираем полносвязный слой. Заменяем слой пулинга своим с размерностью 256 на 256; теперь для генерации описания можно подавать на вход изображение любого размера.

### Decoder LSTM
В качестве декодера используется LSTM ячейка, на вход которой каждую итерацию будут подаваться ранее сгенерированное слово и взвешенная карта признаков, полученная с помощью модуля мягкого внимания. 

## Обучение
Обучение проводилось с размером батча в 32 штуки и дообучением энкодера на протяжении 31 эпохи; лучший результат достигнут на 17-й. В качестве функции потерь использована кросс энтропия, в качестве оптимизатора – Адам.

Для того чтобы не тратить вычисления на токены <pad>, если в батче попались описания разной длины, используется функция pack_padded_sequence(), которая сортирует описания по длине и вычисляет новые размеры батчей для каждого момента во времени (timestep), чтобы в одном батче обрабатывались части описаний без токена \<pad>

Чтобы бороться с переобучением кроме слоя dropout используем и раннюю остановку. Будем следить за оценкой BLEU-4: после каждой эпохи обучения проводим эпоху валидации; если оценка не улучшается в течение 5 эпох, то понижаем коэффициент скорости обучения; если не видно улучшений на протяжении 20 эпох, то останавливаем обучение.

 
## Оценка
В качестве простой и быстрой метрики выступает BLEU, основанная на сходстве n-грамм (не символов, а слов) и похожая на оценку точности: считается, сколько n-грамм сгенерированного текста присутствует в исходном; чем больше похожих n-грамм, тем лучше оценивается предсказание модели.

Для повышения качества оценивания сгенерированных текстов можно задействовать BERTScore: используются предобученные векторы BERT, каждое слово заменяется вектором, считается косинусное сходство между каждой парой слов, выбирается максимальное, умножается на IDF.

<p align="center" width="100%">
 <img src="https://github.com/Mpkosh/EC_descr_gen/blob/main/imgs/train_val_loss.png" width="40%" > 
<p align="center"><i> График ошибок для обучающей и тестовой выборок по эпохам</i></p>
</p>  

Если посмотреть на график ошибок, то можно заметить, что модель переобучается. Так как используется early stopping, следящий за оценкой BLEU-4, то в качестве эпохи, показавшей наилучшие результаты, была выбрана 17-ая, хотя ошибка на тестовой выборке перестала падать с 10-й эпохи.

<p align="center" width="100%">
 <img src="https://github.com/Mpkosh/EC_descr_gen/blob/main/imgs/bleu_n_score.png" width="40%" > 
 <img src="https://github.com/Mpkosh/EC_descr_gen/blob/main/imgs/bertscore_bleu.png" width="40%" >
<p align="center"><i>График оценок BLEU и BERTScore по эпохам</i></p>
</p>  

Как описывается в статье [Improving Image Captioning with Language Modeling Regularizations](10.1109/ASYU48272.2019.8946376), существует «loss-evaluation mismatch», так как во время обучения мы считаем ошибку на уровне слов, а во время оценивания пытаемся улучшить метрику (BLEU-4) на уровне предложений. На рисунке можно увидеть, что с 10-й эпохи, когда ошибка на тестовой выборке увеличивалась, BLEU-4 (вместе от BLEU-1 до BLEU-3) и BERTScore продолжали улучшаться. Можно предположить, что модель начинает предсказывать лучше, но менее уверенно.

После 17 эпохи получены следующие результаты:
* BLEU-4 – 0.066, 
* BERTScore – 0.133. 
 
Часто авторы статей представляют результаты BLEU-4 в процентах, то есть наш итог – 6,6. Считается, что довольно хорошим можно считать результат от 20 и выше. Однако, если посмотреть на результаты в статье [Fashion Captioning: Towards Generating Accurate Descriptions with Semantic Rewards]( 	
https://doi.org/10.48550/arXiv.2008.02693), можно заметить, что их BLEU-4 равен 6,8. 

## Генерация описаний
### Изображение из тестовой выборки 
Для картинки с леопардовым пиджаком и исходным описанием «glinting goldtone button add a dash of decadence to this sharply tailored blazer patterned with feisty leopard spot» сгенерируем новую подпись с помощью нашей модели и визуализируем внимание модели во время каждого слова.

" \<start> a leopard print and slightly cropped hem add to the charm of this tailored jacket cut from a double face italian silk with a leopard print and leopard pattern that"

<p align="center" width="100%">
 <img src="https://github.com/Mpkosh/EC_descr_gen/blob/main/imgs/test_img_attention.png" width="70%" > 
<p align="center"><i>Внимание модели во время генерации описания пиджака</i></p>
</p>  

Чтобы сгенерировать слова «cropped» и «hem», модель концентрируется на нижней части пиджака; слово «jacket» создано при обращении внимания на верхние края пиджака и пуговицу. Интересно, что при генерации слова «add» и внимании снаружи объекта, при генерации следующего слова модель смотрит внутрь предыдущей области.

### Изображение из интернета

"\<start> a sweetheart neckline and a thigh high slit add to the sleek silhouette of this sleek sheath gown that s perfect for the bride or a a little more than"

<p align="center" width="100%">
 <img src="https://github.com/Mpkosh/EC_descr_gen/blob/main/imgs/internet_img_attention.png" width="70%" > 
<p align="center"><i>Внимание модели во время генерации описания платья</i></p>
</p>  

Опираясь на внимание модели, видно, что она концентрируется на разрезе платья, когда говорит «thigh», и на длине нижней части платья, когда говорит «gown».

