# Анализ слайсера Skeinforge
В данном репозитории содержится обзор функционала слайсера Skeinforge и его кода.
## Использование слайсера 
Skeinforge (далее СФ) содержит большое количество настроек для слайсинга и печати, - от настройки толщины слайса вплоть до настроек формы и угла поворота при заполнении.
Загрузим модельку кораблика Бенчи со стандартными параметрами: 
![image](https://github.com/Zysperro/SkeinForgeAnalysis/assets/110100353/f1ba193e-c572-453a-ae7f-bdf92bfb9d21)

На картинке выше виден 5 слой слайснутой модели. Стандартно используется паттерн заполнения прямыми линиями, что видно на картинке.
Среди слоев даже затесалась реклама:
![image](https://github.com/Zysperro/SkeinForgeAnalysis/assets/110100353/554eed10-379e-434b-aef9-10722e08eea3)

При использовании параметра Infill Pattern: Grid Hexagonal, заполнение внутренних полостей будет происходить не сплошными линиями, а шестиугольниками (гексагонами):

![image](https://github.com/Zysperro/SkeinForgeAnalysis/assets/110100353/63a9bc21-3688-4757-84e6-ce682c0df866)
![image](https://github.com/Zysperro/SkeinForgeAnalysis/assets/110100353/d5d325ed-46f4-4fdc-9ff7-5b79ae0da01c)

> <sub>Полученный gcode файл - в репозитории (3DBenchy_export.gcode)</sub>

Основные инструменты программы для слайсинга находятся в папке skeinforge_plugins/craft_plugins, рассмотрим в параметрах профиля порядок создания gcode модели, мы рассматриваем печать 3Д принтеорм, поэтому заглянем в модуль extrusion.py в profile_plugins:
![image](https://github.com/Zysperro/SkeinForgeAnalysis/assets/110100353/3a45b6fc-d579-41c7-bd88-132bf8022eb6)

Порядок следующий:
> carve scale bottom preface widen inset fill multiply speed temperature raft skirt chamber tower jitter clip smooth stretch skin comb cool hop wipe oozebane dwindle splodge home lash fillet limit unpause dimension alteration export

Рассмотрим эти модули по отдельности:

### Carve
Carve - это самый важный плагин, который необходимо настроить для вашего принтера. Он вырезает форму в слои svg слайсов. Он также задает высоту слоя и ширину края для остальных инструментов.

### Scale
Scale масштабирует карвинг(вырезку?), чтобы компенсировать усадку после охлаждения экструзии (материала).
По сути код просто проходится по слоям и умножает координаты на указанный масштаб (стандартно 1.01 для XY и 1 для Z):
![image](https://github.com/Zysperro/SkeinForgeAnalysis/assets/110100353/bb258eb0-ee85-4ca3-924d-0d82f711c1a0)

Сам цикл:
![image](https://github.com/Zysperro/SkeinForgeAnalysis/assets/110100353/d32204dc-3c2b-400b-9305-84f09dc321a0)

### Bottom
Bottom устанавливает дно вырезки на заданную высоту.
Здесь также просто добавляется дельта высоты (для чего находить min между максимальным числом и z - непонятно, возможно чтобы число было не больше максимума):

![image](https://github.com/Zysperro/SkeinForgeAnalysis/assets/110100353/8e3be56d-2661-4c61-a148-78a5ad78b32a)

### Preface
Preface преобразует svg-фрагменты в экструзионные слои gcode, опционально с командами home, positioning, turn off и unit.
Preface содержит собирает gcode, добавление команды выключение экструдера
![image](https://github.com/Zysperro/SkeinForgeAnalysis/assets/110100353/ee35c01a-9d6f-40c5-9221-feaa0d2aa218)

Добавление preface на слои:
![image](https://github.com/Zysperro/SkeinForgeAnalysis/assets/110100353/d0dcbbb7-dede-483e-978e-64fd7fbfbc1e)

### Widen
Расширитель расширяет внешние края по отношению к внутренним, так что внешние края будут отдалены от внутренних по крайней мере на две ширины, и поэтому внешние нити не будут перекрывать внутренние нити.
Например, если кружка имеет очень тонкую стенку, то расширитель расширит внешнюю сторону кружки так, что стенка кружки будет иметь ширину в два края, и нить внешней стенки не будет пересекаться со внутренней.
Или, например, если внешняя сторона объекта проходит рядом с отверстием, то расширитель расширит стенки вокруг отверстия так, что стенка будет выпуклой вокруг отверстия, а внешняя нить не будет перекрывать нить отверстия.
В основном методе addWiden код проходится по всем лупам слоя и находит пересечения 
![image](https://github.com/Zysperro/SkeinForgeAnalysis/assets/110100353/24fff58b-4919-466f-950b-b7ca16b4a005)

Метод для нахождения пересечениий:
![image](https://github.com/Zysperro/SkeinForgeAnalysis/assets/110100353/d238a580-63a8-4836-a778-7d4c40484aa5)

Для нахождения используются метод из библиотеки fabmetheus_utilities - euclidean.isPathInsideLoop

Далее пересекаемые пути объединияются:
![image](https://github.com/Zysperro/SkeinForgeAnalysis/assets/110100353/11424028-0660-4feb-84c8-4568f8673819)

