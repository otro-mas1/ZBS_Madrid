---
jupytext:
  encoding: '# -*- coding: utf-8 -*-'
  formats: ipynb,py:nomarker,md:myst
  text_representation:
    extension: .md
    format_name: myst
    format_version: 0.13
    jupytext_version: 1.10.3
kernelspec:
  display_name: Python pyviz_37
  language: python
  name: pyviz_37
---

<font color='darkblue' size=18>Zonas Basicas de Salud con python</font><img src="img/Mercator-projection.jpg" align='right'><img src="img/python-logo.png" align='right'>

### <font color='darkblue' size=12> Comunidad de Madrid</font>

#### Vamos a usar python y alguna de sus librerías para crear una representación gráfica de datos sobre un mapa.

##### Para ello necesitamos:

    1. Un mapa base.
    2. Shape de cada área del mapa.
    3. Datos a representar de cada área/shape.
    4. Asociar los datos de ZBS a cada shape.
    5. Mapa con los datos.

---
### Después de Crear el mapa interactivo:
- Además de usarlo en el propio Notebook.
- Lo exportamos en un fichero html.
- lo exportamos en fichero gif.

#### En este caso, serán datos de situación del COVID-19 en las Zonas Básicas de Salud de la Comunidad de Madrid.
### Necesitamos datos:
- Los shapes de las Zonas Básicas de Salud que forman el mapa de la Comunidad de Madrid.
- Un dato a representar en el mapa. He escogido la Tasa de Incidencia Acumulada en los últimos 14 días. Esta serie de datos se llamará **TIA_14d**.

### Necesitamos librerías que nos permitan acceder a los datos, procesarlos y presentarlos de manera interactiva:
- Podemos usar datos en local ó acceder a ellos a traves de su url pública. En este caso será con datos desde la web de la CCAA de Madrid.
- Vamos a usar holoviews por su sencillez, está muy relacionado con pandas y permite acceder a bokeh.
- Terminando con opciones interactivas.


<br><br>
### Empecemos importando las librerías que vamos a usar

```{code-cell} ipython3
# # Las dos siguientes líneas de código nos permite usar el 100% del ancho de pantalla.
from IPython.display import display, HTML, Markdown
display(HTML("<style>.container { width:100% !important; }</style>"))
```

```{code-cell} ipython3
from IPython.core.interactiveshell import InteractiveShell  # Para mostrar más de una salida por celda en el notebook.
```

```{code-cell} ipython3
import os
import pathlib
import imageio
import ffmpeg

# Adaptamos el nombre de los 4 últimos ficheros para que ffmpeg les dedique más tiempo.
from pathlib import Path 
from shutil import copyfile, 
import geopandas as gpd
import geoviews as gv
import cartopy.crs as ccrs
import pandas as pd
import numpy as np
import panel as pn
from bokeh.models import FixedTicker as bk_FixedTicker, HoverTool as bk_HoverTool
import holoviews as hv
```

```{code-cell} ipython3
import matplotlib.pyplot as plt
import matplotlib as  mpl
import matplotlib.colors as colors
```

```{code-cell} ipython3
gv.extension('bokeh')
```

<br><br>
## 1. Mapa Base.

    Será de Wikipedia. Hay otras opciones posibles de Mapa Base.

```{code-cell} ipython3
Mapa_Base = gv.tile_sources.Wikipedia
Mapa_Base  # Ya tenemos Mapa Base y es interactivo!!
```

---
---
<br><br>
## 2. Colección de shapes de ZBS de la Comunidad de Madrid.


### - Cargamos en un GeoDataFrame los shapes y una columna extra pob_pad19. Los ficheros los proporciona la Comunidad de Madrid <img src="img/logo_comunidad.png" align='right' />
- La CAM proporciona un fichero zip con los shapes de las Zonas Básicas Sanitarias. [Datos de shapes](https://datos.comunidad.madrid/catalogo/dataset/b3d55e40-8263-4c0b-827d-2bb23b5e7bab/resource/f1837bd3-a835-4110-9bbf-fae06c99b56b/download/zonas_basicas_salud.zip)

[WEB Datos de Zonas Básicas de Madrid](https://datos.comunidad.madrid/catalogo/dataset/covid19_tia_zonas_basicas_salud)
- Podríamos acceder a estos ficheros directamente online ó descargarlo y usarlo como fichero local.
<br><br>
#### **PREVIAMENTE** he descargado esos ficheros a un subdirectorio llamado 'zonas_basicas_salud'
### Cargando los shapes

```{code-cell} ipython3
gdf_CAM = gpd.read_file('zonas_basicas_salud') # Le he indicado el nombre del subdirectorio y geopandas se encarga de cargar lo necesario.
```

```{code-cell} ipython3
gdf_CAM_Mercator = gdf_CAM.to_crs(ccrs.GOOGLE_MERCATOR.proj4_init)  # Hay que adaptar las coordenadas de los shapes al estandar Mercator. Conservo los dos GeoDataFrame por si se quieren ver después.
```

```{code-cell} ipython3
Mapa_ZBS = gv.Polygons(gdf_CAM_Mercator, vdims=['pob_pad19', 'zona_basic'], crs=ccrs.GOOGLE_MERCATOR)  
Mapa_ZBS   # Ya tenemos la capa de ZBS!!
```

### 2.1 Juntamos el Base y los Shapes de ZBS

```{code-cell} ipython3
Mapa_Base * Mapa_ZBS  # Juntando Piezas.
```

----
----
<br><br>
## 3. Cargamos datos de COVID para cada ZBS

- La CAM proporciona un fichero csv con los datos COVID por Zona Básica Sanitaria. Se actualiza aproximadamente, una vez por semana, y suele ser los martes.  [Datos COVID-19 por Zona Básica Sanitaria](https://datos.comunidad.madrid/catalogo/dataset/b3d55e40-8263-4c0b-827d-2bb23b5e7bab/resource/43708c23-2b77-48fd-9986-fa97691a2d59/download/covid19_tia_zonas_basicas_salud_s.csv)

```{code-cell} ipython3
# Cargamos los datos y los preparamos un poquito.

url_fichero="https://datos.comunidad.madrid/catalogo/dataset/b3d55e40-8263-4c0b-827d-2bb23b5e7bab/resource/43708c23-2b77-48fd-9986-fa97691a2d59/download/covid19_tia_zonas_basicas_salud_s.csv"

df_zonas_basicas = pd.read_csv(url_fichero, sep=';', encoding='iso8859_15', decimal=',', parse_dates=['fecha_informe'])

df_zonas_basicas.rename(columns={'tasa_incidencia_acumulada_ultimos_14dias':'TIA_14d'}, inplace=True)  # Cambio el nombre de la columna por comodidad posterior.
Markdown(f"<font size=12>Datos hasta: <font color='darkred'>{df_zonas_basicas.fecha_informe.max().strftime('%F')}")
```

```{code-cell} ipython3
df_zonas_basicas['codigo_geo']=df_zonas_basicas['codigo_geometria'].str.strip()  # Los codigos vienen con espacios por detrás. Se los quito y aprovecho para cambiar el nombre.
df_zonas_basicas['TIA_14d']=df_zonas_basicas['TIA_14d'].astype('int')
df_zonas_basicas.drop(columns='codigo_geometria', inplace=True)  # Ya no es necesaria esta columna.
```

```{code-cell} ipython3
df_zonas_basicas.sort_values(by='fecha_informe', ascending=False).head()
```

<br><br>
### 3.1 Filtramos los últimos datos del fichero

+++

 En el fichero vienen los datos semanales desde el 2 de julio. Ahora, queremos el último dato de cada ZBS. 

```{code-cell} ipython3
df_zbs_ultimo_dia = df_zonas_basicas.loc[(df_zonas_basicas.fecha_informe==df_zonas_basicas.fecha_informe.max()),['TIA_14d','codigo_geo','fecha_informe']]
```

```{code-cell} ipython3
df_zbs_ultimo_dia['fecha_informe']=df_zbs_ultimo_dia.fecha_informe.dt.strftime('%Y-%m-%d')
```

```{code-cell} ipython3
df_zbs_ultimo_dia.info(memory_usage='deep')
```

<br><br>
Vemos un DataFrame con 286 registros.

Uno por cada ZBS
Con 3 campos en cada registro.
- Codigo_geo:     Identifica a cada ZBS.
- fecha_informe:  fecha de los datos
- TIA-14:         Tasa de Incidencia Acumulada en los últimos 14 días.

```{code-cell} ipython3
df_zbs_ultimo_dia.head(3)
```

----
----

<br><br>
## 4. Añadimos los datos de TIA_14d al GeoDataFrame

```{code-cell} ipython3
gdf_CAM_COVID=gdf_CAM_Mercator.merge(df_zbs_ultimo_dia,  on='codigo_geo')
gdf_CAM_COVID.info(memory_usage='deep')
```

```{code-cell} ipython3
Mapa_ZBS_COVID = gv.Polygons(gdf_CAM_COVID, vdims=['TIA_14d',('zona_basic', 'Zona Básica de Salud'), ('fecha_informe', 'Fecha del Dato'), 'pob_pad19'], crs=ccrs.GOOGLE_MERCATOR).opts( line_color='grey', cmap='magma_r')
Mapa_ZBS_COVID
```

---
---
### 5. Mapa interactivo con los datos.

 - Juntamos Mapa Base y la capa Mapa_ZBS_COVID.
 - Tocamos algo las opciones para adaptar la visibilidad.

```{code-cell} ipython3
df_zbs_ultimo_dia.info(memory_usage='deep')
```

Definimos un colormap a medida para la ocasión con 5 Niveles de Color y añadimos el negro.

```{code-cell} ipython3
colormap_COVID = ['#006837' , '#86cb66', '#ffffbf', '#f98e52', '#FF2222','#a50026']
```

```{code-cell} ipython3
df_zonas_basicas.groupby('fecha_informe').agg({'TIA_14d':['median', 'sum','describe']}).tail()
```

```{code-cell} ipython3
df_zbs_ultimo_dia.sort_values(by='TIA_14d', ascending=True)
```

```{code-cell} ipython3
Niveles_COVID = [0, 25, 50, 150, 250, 500, 530]
```

```{code-cell} ipython3
Colorbar_ticks = bk_FixedTicker(ticks=[  12.5, 25, 37.5, 50, 100, 150, 200, 250, 325, 500, 530])
```

```{code-cell} ipython3
Colorbar_labels =  {
            37.5: 'Riesgo Bajo',
            12.5: 'Nueva Normalidad',
            25:   '25',
            50:   '50',
            100: 'Riesgo Medio',
            150:  '150',
            200: 'Riesgo Alto',
            250:  '250',
            325: 'Riesgo Extremo',
            500: '500 - Confinamiento Recomendado ECDC',
            530: ''
        }
```

# Mapa Base con Shapes de ZBS y con gradiente de color según TIA_14d

```{code-cell} ipython3
Mapa_Base.opts(alpha=0.7, xaxis=None, yaxis=None) * \
Mapa_ZBS_COVID.opts( color='TIA_14d',  width=900, xaxis=None, yaxis=None, height=800, tools=['hover'], alpha=0.75,
                    color_levels=Niveles_COVID, clim=(0,530), 
                    cmap= colormap_COVID, colorbar=True, colorbar_opts={
                    'major_label_overrides': Colorbar_labels,
                    'major_label_text_align': 'left', 'ticker': Colorbar_ticks
                    })
```

# Hacemos otro?

Fácil de adaptar, el tamaño, la transparencia ('alpha') de los shapes. 

```{code-cell} ipython3
Mapa_ZBS_COVID.opts( width=700,height=500, alpha=0.95)
```

----
<br><br>
## Y cuando lo exportamos en formato html. Sigue interactivo!!

```{code-cell} ipython3
gv.renderer('bokeh').save( Mapa_ZBS_COVID, 'ZBS_Madrid')  # aquí hemos creado el fichero ZBS_Madrid.html SIN MAPA BASE
```

Aqui lo exportamos CON MAPA BASE.
Personalmente me parece mejor opción.
Cuando haces zoom con la herramienta,
te permite ir identificando por donde te mueves en el mapa.

```{code-cell} ipython3
gv.renderer('bokeh').save(Mapa_Base.opts(alpha=0.7, xaxis=None, yaxis=None) * \
                    Mapa_ZBS_COVID.opts(\
                        color='TIA_14d',  width=900, xaxis=None, yaxis=None, height=800, tools=['hover'], alpha=0.75,
                        color_levels=Niveles_COVID, clim=(0,530), 
                        cmap= colormap_COVID, colorbar=True, colorbar_opts={
                        'major_label_overrides': Colorbar_labels,
                        'major_label_text_align': 'left', 'ticker': Colorbar_ticks
                        }), 'ZBS_Madrid_CON_Base') 
```

<br><br>

## Lo hacemos más interactivo con opción de Vistas por Condición?

    - Por ejemplo: Filtrar las ZBS con TIA_14d mayor que un valor indicado. Es fácil.
    
    AVISO: Tarda un poquito en realizar el filtrado.

+++

Creamos un pop-up a medida

```{code-cell} ipython3
hover_ZBS = bk_HoverTool(tooltips= [  ("Zona Básica Sanitaria  ", "  @zona_basic"), ("TIA-14 ", " @{TIA_14d}{0.0}"), ("Dia " , " @{fecha_informe}{%F}") ],
                         formatters={'@{fecha_informe}': 'datetime'} )
```

Creamos una función para reutilizarla

```{code-cell} ipython3
gv.extension('bokeh')
```

```{code-cell} ipython3
def Actualizar_polys(umbral):
    
    return gv.Polygons(gdf_CAM_COVID[gdf_CAM_COVID['TIA_14d'] > umbral],
                       vdims=['TIA_14d', 'zona_basic','fecha_informe'],
                       crs=ccrs.GOOGLE_MERCATOR).opts(\
                                clim=(0, 530), width=850, height=800,
                                tools=[hover_ZBS], alpha=0.75,
                                color_levels=Niveles_COVID,
                                line_color='grey',
                                cmap= colormap_COVID,
                                colorbar=True, colorbar_opts={
                                    'major_label_overrides': Colorbar_labels,
                                    'major_label_text_align': 'left', 'ticker': Colorbar_ticks
                                    }
                    )
```

```{code-cell} ipython3
wdg_IntSlider = pn.widgets.IntSlider(name='TIA_14d Mayor que ', start=0, end=int(gdf_CAM_COVID['TIA_14d'].max()-1), value=250)
```

```{code-cell} ipython3
pn.Row(
    wdg_IntSlider,
    gv.tile_sources.Wikipedia(alpha=0.6) * 
    gv.DynamicMap(pn.bind(Actualizar_polys, wdg_IntSlider))
)
```

---
---

<br><br>
## Ahora creamos un GeoDataFrame con los datos históricos de las ZBS
## para todas las fechas desde el verano en el GeoDataFrame gdf_zbs_historico.

```{code-cell} ipython3
gdf_CAM.crs
```

```{code-cell} ipython3
gdf_zbs_historico = gdf_CAM_Mercator.merge(df_zonas_basicas, on='codigo_geo')
```

```{code-cell} ipython3
gdf_zbs_historico.info(memory_usage='deep')
```

<br><br>
# <font color='darkblue' size=12>Más Interactivo
## Permitimos interactuar tanto por umbral de TIA_14d
## como la evolución en el tiempo de todas las ZBS.

```{code-cell} ipython3
hv.extension( 'bokeh')
```

```{code-cell} ipython3
gv.extension('bokeh')
def Actualizar_polys_fecha(umbral, fecha):
    
    return gv.Polygons(gdf_zbs_historico.loc[(gdf_zbs_historico['TIA_14d'] > umbral) & (gdf_zbs_historico.fecha_informe.dt.strftime("%F")==fecha)],
                       vdims=['TIA_14d', 'zona_basic', 'fecha_informe'], crs=ccrs.GOOGLE_MERCATOR).opts(
                            clim=(0, 530), width=800, height=650, tools=[hover_ZBS], xaxis=True, yaxis=True,
                            color_levels=Niveles_COVID, line_color='grey',
                            cmap= colormap_COVID, colorbar=True, colorbar_opts={
                            'major_label_overrides': Colorbar_labels,
                            'major_label_text_align': 'left', 'ticker': Colorbar_ticks, 'scale_alpha':0.7
                            }, alpha=0.70 )
```

```{code-cell} ipython3
fechas = sorted(gdf_zbs_historico.fecha_informe.dt.strftime("%F").unique())
```

```{code-cell} ipython3
wdg_IntSlider = pn.widgets.IntSlider(name='TIA_14d Mayor que ', start=0, end=int(gdf_zbs_historico['TIA_14d'].max()-1))
wdg_Slider = pn.widgets.DiscreteSlider(name='Dia ', options=fechas, value=fechas[-1])
```

```{code-cell} ipython3
pn.Row( pn.Column(
            Markdown(f"## <br><br> Puedes filtrar el valor de TIA-14d."),
            Markdown(f"### Presentará las ZBS con TIA_14d Mayor de la seleccionada."),
            Markdown(f"<br>## Se puede seleccionar otras fechas para ver el pasado y la evolución"),
    wdg_IntSlider, wdg_Slider),
    gv.tile_sources.Wikipedia(alpha=0.99) * # Un valor tan alto, no afecta. Simplemente recuerda ser cambiable.
    gv.DynamicMap(pn.bind(Actualizar_polys_fecha, wdg_IntSlider, wdg_Slider))
)
```

```{code-cell} ipython3
# Cuántás ZBS hay en cada Nivel de Riesgo
df_rangos = gdf_CAM_COVID.TIA_14d.value_counts(bins=pd.IntervalIndex.from_tuples([ (0, 25), (25,50), (50, 150), (150, 250), (250, 400), (400, gdf_CAM_COVID.TIA_14d.max()+1)], closed='left' )).sort_index(ascending=False)

df_rangos
```

---
<br><br>
#### Vamos a crear la evolución de la IA-14d para la CAM en formato gif.
#### Para ello, primero creamos el Mapa de la CAM para una fecha determinada y lo exportamos a un fichero png.
#### Luego creamos un bucle por todas las fechas y exportamos cada uno de los ficheros png.
#### Terminamos juntando todos los ficheros png en un fichero gif.

```{code-cell} ipython3
Colorbar_ticks = bk_FixedTicker(ticks=[  12.5, 25, 37.5, 50, 100, 150, 200, 250, 325, 500, 530])
```

```{code-cell} ipython3
mpl_ticks=[  12.5, 25, 37.5, 50, 100, 150, 200, 250, 325, 500, 530]
Colorbar_labels =  {
            37.5: 'Riesgo Bajo',
            12.5: 'Nueva Normalidad',
            25:   '25',
            50:   '50',
            100: 'Riesgo Medio',
            150:  '150',
            200: 'Riesgo Alto',
            250:  '250',
            325: 'Riesgo Extremo',
            500: '500 - Confinamiento Recomendado ECDC',
            530: ''
        }
```

### Creamos una función con Matplotlib
### - creará el mapa para cada fecha
### - Lo graba en fichero png
### Después los juntaremos en un fichero gif

```{code-cell} ipython3
# Rango de valores
vmin, vmax = 0, 530
hv.extension( 'matplotlib');
```

```{code-cell} ipython3
## Función Mapa_ZBS_para fecha.
def mpl_ZBS_fecha(gdf, vmin, vmax, fecha, titulo):
    
    # colormap a medida
    cmap = colors.ListedColormap(colormap_COVID, N=6)
    norm = colors.BoundaryNorm(Niveles_COVID, len(Niveles_COVID))

    fig, ax = plt.subplots( figsize=(10, 9) )
    
    mpl_mapa = gdf.plot( column='TIA_14d',cmap=cmap, ax=ax,
                        linewidth=0.1, edgecolor='0.8', 
                        vmin=vmin, vmax=vmax, norm=norm )

    mpl_mapa.set_title(titulo, fontdict={'fontsize': 20, 'fontweight' : 3})
    
    mpl_mapa.axis('off')

    mpl_mapa.annotate(gdf.fecha_informe.iloc[0].strftime("%F"),
                xy=(0,0),xytext=(480,420), xycoords='figure points', fontsize= 25 )

    cax = fig.add_axes([0.1, 0.1, 0.051, 0.75])  # Para ubicar colorbar

    cbar =plt.colorbar(
        mpl.cm.ScalarMappable(cmap=cmap, norm=norm),
        cax=cax,
        boundaries= Niveles_COVID,
        extend='neither',
        spacing='proportional',
        orientation='vertical',
        ticks=mpl_ticks,
    )

    
    cbar.set_ticklabels(['Nueva Normalidad', 25, 'Riesgo Bajo', 50, 'Riesgo Medio', 150, 'Riesgo Alto', 250, 'Riesgo Extremo', '+500 Confinamiento\nRecomendado ECDC'] )

    bbox_inches='tight'
    
    return fig
```

Ahora probamos la creación de un Mapa para una fecha con los datos para una fecha concreta

```{code-cell} ipython3
fig_Mapa_ZBS_fecha =    mpl_ZBS_fecha(gdf_zbs_historico.loc[(gdf_zbs_historico.fecha_informe.dt.strftime("%F")=="2021-04-06")],
                                   vmin, vmax, 
                                   '2021-04-06',
                                   'Comunidad de Madrid -- Semáforo COVID\n Incidencia Acumulada en 14 días por 100.000 habitantes')
plt.show()
plt.close(fig_Mapa_ZBS_fecha)
```

La salida del mapa puede parecer desajustada.
Sin embargo, al crear el gif lo deja 'colocado'

+++

Ahora usamos la funcion para crear un Mapa para cada fecha

```{code-cell} ipython3
# Antes de Crear los mapas de cada semana.
# Borramos los mapas existentes de ejecuciones anteriores
#
for f in Path('img_temp/').glob('20*.png'):
    try:
        f.unlink()
    except OSError as e:
        print("Error: %s : %s" % (f, e.strerror))
```

```{code-cell} ipython3
lista_files=[]
hv.extension( 'matplotlib');

for fecha in gdf_zbs_historico.fecha_informe.dt.date.unique():  # Para cada fecha
    
    gdf_zbs_fecha = gdf_zbs_historico.loc[(gdf_zbs_historico.fecha_informe.dt.strftime("%F")==fecha.strftime("%F"))]
    
    png_fecha=f"img_temp/{gdf_zbs_fecha.fecha_informe.iloc[0].strftime('%F')}.png"

    print(f"Creando para {fecha} ... fichero {png_fecha}")
    
    fig_Mapa_ZBS_fecha = mpl_ZBS_fecha( gdf_zbs_fecha, vmin, vmax, fecha,
                                  'Comunidad de Madrid -- Semáforo COVID\n Incidencia Acumulada en 14 días por 100.000 habitantes')
    
    fig_Mapa_ZBS_fecha.savefig(png_fecha, dpi=100, facecolor='lightgray', pad_inches=0.1  )
    
    plt.close(fig_Mapa_ZBS_fecha)

    lista_files.append(png_fecha)  # Añadido a la lista de ficheros.
```

---
<br>
<br>

### Ahora hacemos la conversión a gif de los png generados

```{code-cell} ipython3
# Duplicamos los últimos mapas para poder verlos mejor
# en el gif y en el mp4.

images = []   # Lista de las imágenes
frames=sorted(lista_files[:20])  # Damos más tiempo a los más recientes

for I in range(0,5):
    for _ in frames[-1*I:]:
        #ff_t = lista_files[-1*I]
        print(_, Path(_).parts[0]+"/"+Path(_).stem+chr(97+I)+".png")
        copyfile( _, Path(_).parts[0]+"/"+Path(_).stem+chr(97+I)+".png")

        
        
        
# Vamos con la creación del gif        

       
for file_name in sorted(frames):        
        images.append(imageio.imread(file_name))

        
imageio.mimsave('img/CAM_ia14d.gif', images, fps=1 )
```

```{code-cell} ipython3

# ---
# <br>
# <br>
# <font color='darkblue' size=18 > Terminamos creando un video, con ffmpeg<br>
# <br>
#
out, _ = (
    ffmpeg 
    .input('img_temp/2021*.png', pattern_type='glob', framerate=1.2) 
    .output('ia14_CAM1.mp4') 
    .overwrite_output() 
    .run()
)
```

```{code-cell} ipython3
df_zonas_basicas.set_index('fecha_informe', drop=True, append=False).loc['2021-03-23'].TIA_14d.hist(alpha=0.5,  color='red', label='2021-03-30')
df_zonas_basicas.set_index('fecha_informe', drop=True, append=False).loc['2021-03-30'].TIA_14d.hist(alpha=0.5, color='blue', label='2021-04-06')
plt.legend(['semana pasada', 'semana actual'], fontsize=12)
plt.grid(b=True, which='major', axis='y', linestyle='dotted')
plt.grid(b=False, which='major', axis='x')
plt.title("ZBS Madrid, Incidencia Acumulada 14d", fontsize=16);
```

```{code-cell} ipython3

```

```{code-cell} ipython3
def zbs_pos_semana(x: pd.DataFrame()):
    x=x.sort_values(by='TIA_14d', ascending=True).copy()
    x['Pos_semanal']=np.arange(len(x))
    return x
df_t = df_zonas_basicas.set_index('str_fecha_informe',drop=True, append=False).xs([  'zona_basica_salud', 'TIA_14d'], axis=1).groupby(['str_fecha_informe'], as_index=False).apply(zbs_pos_semana)
df_t.info(memory_usage='deep')
#df_t.drop(level=0,  axis=0)
df_t.droplevel(level=0).info(memory_usage='deep')
```
