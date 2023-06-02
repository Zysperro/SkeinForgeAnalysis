# Анализ слайсера Skeinforge
В данном репозитории содержится исходный код программы Skeinforge, а в данном описании - мой обзор функционала слайсера и его кода
## Использование слайсера 
Skeinforge (далее СФ) содержит большое количество настроек для слайсинга и печати, - от настройки толщины слайса вплоть до настроек формы и угла поворота при заполнении.
Загрузим модельку кораблика Бенчи со стандартными параметрами: 
![Laayer5](https://github.com/Zysperro/SkeinForgeAnalysis/assets/110100353/e4293420-7763-414c-98e2-c89add1d3bf5)

На картинке выше виден 5 слой слайснутой модели. Стандартно используется паттерн заполнения прямыми линиями, что видно на картинке.
Среди слоев даже затесалась реклама:
![AdLayer](https://github.com/Zysperro/SkeinForgeAnalysis/assets/110100353/8840d919-6171-4a91-a19c-258db8349e59)

При использовании параметра Infill Pattern: Grid Hexagonal, заполнение внутренних полостей будет происходить не сплошными линиями, а шестиугольниками (гексагонами):

![HaxaFill](https://github.com/Zysperro/SkeinForgeAnalysis/assets/110100353/7ba821c8-8e02-469e-8071-e6572610faf0)
![Hexa](https://github.com/Zysperro/SkeinForgeAnalysis/assets/110100353/525987ac-0a65-4d2f-96fe-ed7a330ee423)

Основные инструменты программы для слайсинга находятся в папке skeinforge_plugins/craft_plugins, рассмотрим в параметрах профиля порядок создания gcode модели, мы рассматриваем печать 3Д принтеорм, поэтому заглянем в модуль extrusion.py в profile_plugins:
![image](https://github.com/Zysperro/SkeinForgeAnalysis/assets/110100353/13e01bad-1bfc-4612-80b9-ad2dd68f77e0)

Порядок следующий:
> carve scale bottom preface widen inset fill multiply speed temperature raft skirt chamber tower jitter clip smooth stretch skin comb cool hop wipe oozebane dwindle splodge home lash fillet limit unpause dimension alteration export

Рассмотрим эти модули по отдельности:

### Carve
Carve - это самый важный плагин, который необходимо настроить для вашего принтера. Он вырезает форму в слои svg слайсов. Он также задает высоту слоя и ширину края для остальных инструментов.

### Scale
Scale масштабирует карвинг(вырезку?), чтобы компенсировать усадку после охлаждения экструзии (материала).
По сути код просто проходится по слоям и умножает координаты на указанный масштаб (стандартно 1.01 для XY и 1 для Z):
![image](https://github.com/Zysperro/SkeinForgeAnalysis/assets/110100353/d7a3fbb3-9001-46b8-b0f1-bc7c09695b41)

Сам цикл:
![image](https://github.com/Zysperro/SkeinForgeAnalysis/assets/110100353/f04b7134-0698-4075-bb55-3a4a0abdd56e)

### Bottom
Bottom устанавливает дно вырезки на заданную высоту.
Здесь также просто добавляется дельта высоты (для чего находить min между максимальным числом и z - непонятно, возможно чтобы число было не больше максимума):

![image](https://github.com/Zysperro/SkeinForgeAnalysis/assets/110100353/8d990af6-30e3-4477-8a1e-e26952469053)

### Preface
Preface преобразует svg-фрагменты в экструзионные слои gcode, опционально с командами home, positioning, turn off и unit.
Preface содержит собирает gcode, добавление команды выключение экструдера
![image](https://github.com/Zysperro/SkeinForgeAnalysis/assets/110100353/464f8882-2e4c-48bb-a1ea-8b5650f22235)

Добавление preface на слои:
![image](https://github.com/Zysperro/SkeinForgeAnalysis/assets/110100353/f2c58054-bd42-4acb-a597-6548dc3b5e83)

### Widen
Расширитель расширяет внешние края по отношению к внутренним, так что внешние края будут отдалены от внутренних по крайней мере на две ширины, и поэтому внешние нити не будут перекрывать внутренние нити.
Например, если кружка имеет очень тонкую стенку, то расширитель расширит внешнюю сторону кружки так, что стенка кружки будет иметь ширину в два края, и нить внешней стенки не будет пересекаться со внутренней.
Или, например, если внешняя сторона объекта проходит рядом с отверстием, то расширитель расширит стенки вокруг отверстия так, что стенка будет выпуклой вокруг отверстия, а внешняя нить не будет перекрывать нить отверстия.
В основном методе addWiden код проходится по всем лупам слоя и находит пересечения 
![image](https://github.com/Zysperro/SkeinForgeAnalysis/assets/110100353/f09751e4-230d-43aa-a16e-f082d8b20eb4)

Метод для нахождения пересечениий:
![image](https://github.com/Zysperro/SkeinForgeAnalysis/assets/110100353/0da92455-39ab-4809-979f-9d61087cb00a)

Для нахождения используются метод из библиотеки fabmetheus_utilities - euclidean.isPathInsideLoop

Далее пересекаемые пути объединияются:
![image](https://github.com/Zysperro/SkeinForgeAnalysis/assets/110100353/dbf9317a-fd73-4e7f-93ec-a36038c66918)

