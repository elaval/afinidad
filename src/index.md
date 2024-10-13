---
toc: false

sql:
  afinidad: data/affinity_details_with_names.parquet
---


# Dime como votas ...
Esta página explora la afinidad entre diputadas y diputados de acuerdo a su comportamiento en 996 votaciones entre Enero y Octubre de 2024.

Por ejemplo, si Diputada Cariola y el Diputado Kaiser han concurrido a 768 votaciones y en 281 han votado de la misma manera (en 263 ambos Afirmativo, en 14 ambos En Contra y en 4 ambos Abstención), entonces decimos que tienen un 36,6% de **afinidad** (en 36,6% de las votaciones votaron de la misma manera).  Y también hay un 63,4% de **distancia** (en 63,4% de las votaciones votan de manera diferente).

## Visualización de la distancia de una diputada o diputado con el resto de sus pares
Si seleccionas una persona, podrás ver gráficamente la distancia (según votaciones) de esa persona con el resto de las o los diputados.
```js
const dipSeleccionado = view(Inputs.select(  
  _.chain(listaDiputados)
    .sortBy((d) => d.ApellidoPaterno)
    .value(),
  {
    label: "Diputada o diputado",
    format: (d) => `${d.Nombre} ${d.ApellidoPaterno} (${d.aliasPartido})`
  }));
```

```sql id=datosAfinidadConDipSeleccionado
SELECT
  CASE WHEN diputado1Id = ${dipSeleccionado.Id} THEN diputado2Id ELSE diputado1Id END AS id,
  CASE WHEN diputado1Id = ${dipSeleccionado.Id} THEN diputado2Name ELSE diputado1Name END AS nombre,
  afinidad,
  distancia
FROM afinidad
WHERE diputado1Id = ${dipSeleccionado.Id} OR diputado2Id = ${dipSeleccionado.Id}


```

<div class="card">
<div>${chart1}</div>
</div><!--card-->



```js
const chart1 = (()=> {
  
  const data = _.concat(
    {
      id: dipSeleccionado.Id,
      nombre: `${dipSeleccionado.Nombre} ${dipSeleccionado.ApellidoPaterno}`,
      afinidad: 1,
      distancia: 0
    },
    [...datosAfinidadConDipSeleccionado]
  );

  const maxDistancia = _.chain(data)
    .map((d) => d.distancia)
    .max()
    .value();

  const radio = width / 55;

  return Plot.plot({
    style: { fontSize: 12 },
        marginTop: 50,

    marginBottom: 50,
    //marginLeft: 0,
    x: { inset: 20, label: "Fecha de nacimiento" },
 y: {
      reverse: true,
      ticks: 10,
      label: "",
      labelArrow: "none",
      tickFormat: "%"
    },
    height: width*3/4,
    width,
    marks: [
      Plot.image(
        data,
        Plot.dodgeX(
          { anchor: "middle" },
          {
            y: "distancia",
            src: (d) => `https://www.camara.cl/img.aspx?prmID=GRCL${d.id}`,
            r: radio,
            preserveAspectRatio: "xMidYMin slice",
            tip: true,
            title: (d) => `${getFullName(d.id)}`
          }
        )
      ),
      Plot.text(
        data,
        Plot.dodgeX(
          { anchor: "middle" },
          {
            y: "distancia",
            text: (d) => `${getFullName(d.id)}`,
            r: radio,
            opacity: (d) => (d.id == dipSeleccionado.Id ? 1 : 0),
            fontSize: 14,
            dy: -radio -10
          }
        )
      ),
      Plot.text([{ distancia: 0, text: "Menor distancia" }], {
        y: "distancia",
        dy: -radio - 10,
        dx: -width / 2,
        text: "text",
        fontSize: width / 30,
        fill: "grey",
        textAnchor: "start"
      }),
      Plot.text([{ distancia: maxDistancia, text: "Mayor distancia" }], {
        y: "distancia",
        dy: radio,
        dx: -width / 2,
        text: "text",
        fontSize: width / 30,
        fill: "grey",
        textAnchor: "start"
      })
    ]
  });

})()

```


## Conglomerados según afinidad
A continuación se muestran conglomerados o agrupaciones de parlamentarios según su cercanía en la manera de votar. Para generar los grupos de parlamentarios, se empleó un enfoque de conglomerados jerárquicos (también conocido como clustering jerárquico). Este método nos permite agrupar a los parlamentarios según sus similitudes en las votaciones, sin necesidad de determinar previamente el número de grupos.

¿Cómo funciona este método?  
  
**Comparación de votaciones**: Primero, se analiza cómo ha votado cada parlamentario en comparación con los demás. Se identifica el grado de similitud entre sus patrones de votación.  
**Agrupación inicial:** Los parlamentarios que tienen patrones de votación más similares se agrupan juntos. Esto se hace de manera progresiva, comenzando por los que son más parecidos.  
**Construcción de una jerarquía**: Estos grupos iniciales se vuelven a comparar entre sí. Los grupos que son similares se unen para formar grupos más grandes. Este proceso se repite, creando una estructura en forma de árbol que muestra cómo se agrupan los parlamentarios en diferentes niveles.  
**Visualización de resultados**: El resultado es una representación que nos permite ver claramente las afinidades y alianzas entre los parlamentarios. Podemos identificar grupos que comparten opiniones similares o que suelen votar de manera parecida.  
¿Por qué es útil este enfoque?  
Esta es una herramienta para analizar y comprender el comportamiento legislativo. EL ejercicio ayuda a revelar patrones ocultos en las votaciones, permitiendo entender mejor las dinámicas políticas y las posibles alianzas.
```js
const numGroups = view(Inputs.radio([2, 5, 19], {
  label: "Número de conglomerados",
  value: 5
}));
```

<div class="card">
<div>${chart2}</div>
</div><!--card-->

```js
const chart2 = (() => {
  const dataWChildren = {
    children: _.chain(dataClustersPorTamaño[numGroups])
      .groupBy((d) => d.cluster)
      .map((items, key) => ({
        cluster: key,
        children: items.map((d) => ({
          id: d.diputadoId,
          diputado: d.diputadoName,
          cluster: d.cluster,
          imageUrl: `https://www.camara.cl/img.aspx?prmID=GRCL${d.diputadoId}`,
          fullName: getFullName(d.diputadoId)
        }))
      }))
      .value()
  };

  return buildCirclePack(dataWChildren, {
    highlight: "Jiles",
    selectClusters: []
  });
})()
```

```js
// Carga lista de diputadas y duputados en el período

const listaDiputados = await (async () => {
  function getPartido(record) {
    const d = record.Militancias.Militancia;

    const recordPartido = !d.length
      ? d
      : _.chain(d)
          .sortBy((d) => d.FechaTermino)
          .last()
          .value();

    return recordPartido.Partido.Alias;
  }

  const diputadosPeriodo = await FileAttachment("./data/diputadosPeriodoActual.json").json();


  const dict = {};
  const listaDiputados = diputadosPeriodo.DiputadosPeriodoColeccion.DiputadoPeriodo.map(
    (d) => d.Diputado
  ).map((d) => {
    d.aliasPartido = getPartido(d);
    return d;
  });

  return listaDiputados
})()

const dictDiputado = (() => {
  const dict = {};

  listaDiputados.forEach(d => {
    dict[d.Id] = d;
  })


  return dict;
})();
```

```js
const dataClustersPorTamaño = ({
  2: await FileAttachment("./data/diputado_clusters_num_2.csv").csv(),
  5: await FileAttachment("./data/diputado_clusters_num_5.csv").csv(),
  19: await FileAttachment("./data/diputado_clusters_num_19.csv").csv()
})
```






```js
function getFullName(id) {
  const record = dictDiputado[id];
  return (
    record &&
    `${record.Nombre} ${record.ApellidoPaterno} (${record.aliasPartido})`
  );
}
```

```js
function buildCirclePack(dataWChildren, { selectClusters = [] , clusterSize=0, searchText = ""}) {
  // Specify the dimensions of the chart.
  //const width = width;
  const height = width * 0.75;
  const margin = 1; // to avoid clipping the root circle stroke

  // Specify the number format for values.
  const format = (d) => `${d3.format(",d")(d)}`;

  // Create the pack layout.
  const pack = d3
    .pack()
    .size([width - margin * 2, height - margin * 2])
    .padding(3);

  const searchRegExp = new RegExp(searchText, "i");

  const dataWChildrenFiltered = {
    diputado: null,
    children: dataWChildren.children.filter((d) =>
      clusterSize ? d.children.length == clusterSize : true
    )
  };

  // Compute the hierarchy from the JSON data; recursively sum the
  // values for each node; sort the tree by descending value; lastly
  // apply the pack layout.
  const root = pack(
    d3
      .hierarchy(dataWChildrenFiltered)
      .sum((d) => 1)
      .sort((a, b) => b.value - a.value)
  );

  //return root;
  //return selectClusters;

  const avatarSize = root.children[0].children[0].r * 2;

  // Create the SVG container.
  const svg = d3
    .create("svg")
    .attr("width", width)
    .attr("height", height)
    .attr("viewBox", [-margin, -margin, width, height])
    .attr("style", "width: 100%; height: auto; font: 10px sans-serif;")
    .attr("text-anchor", "middle");

  svg
    .append("clipPath")
    .attr("id", "clipObj")
    .append("circle")
    .attr("cx", 0)
    .attr("cy", 0)
    .attr("r", avatarSize / 2);

  // Place each node according to the layout’s x and y values.
  const node = svg
    .append("g")
    .selectAll()
    .data(root.descendants())
    .join("g")
    .attr("transform", (d) => `translate(${d.x},${d.y})`)
    .attr("opacity", (d) =>
      selectClusters.length && !selectClusters.includes(+d.data.cluster) ? 0 : 1
    );

  // Add a title.
  node
    .append("title")
    .text((d) =>
      !d.height ? `${getFullName(d.data.id)}` : `Cluster: ${d.data.cluster}`
    );

  // Add a filled or stroked circle.
  node
    .append("circle")
    .attr("fill", (d) => 
     d.depth == 0
        ? "none"
        : d.children
        ? "#fff"
        : searchText && d.data.fullName.match(searchRegExp)
        ? "orange"
        : "")
    .attr("stroke", (d) =>
      d.depth == 0
        ? ""
        : d.children
        ? "#bbb"
        : searchText && d.data.fullName.match(searchRegExp)
        ? "orange"
        : ""
    )
    .attr("stroke-width", (d) =>
      searchText && d.data.fullName && d.data.fullName.match(searchRegExp)
        ? 5
        : 1
    )
    .attr("r", (d) => d.r);

  node
    .append("image")
    .attr("xlink:href", (d) => d.data.imageUrl)
    .attr("x", -avatarSize / 2)
    .attr("y", -avatarSize / 2)
    .attr("width", avatarSize)
    .attr("height", avatarSize)
    .attr("clip-path", "url(#clipObj)");

  // Add a label to leaf nodes.
  const text = node
    .filter((d) => !d.children && d.r > 10)
    .append("text")
    .attr("fill", (d) => "white");

  // Add a tspan for each CamelCase-separated word.
  text
    .selectAll()
    .data((d) => [d.data.fullName])
    .join("tspan")
    .attr("x", 0)
    //.attr("y", (d, i, nodes) => `${i - nodes.length / 2 + 0.35}em`)
    .attr("y", (d) => `${5}em`)
    .style("font-size", (d) => `${avatarSize / 150}em`)
    .text((d) => d);

  return svg.node();
}
```

## Fuente de datos
Datos abiertos publicados por la Cámara de Diputados y Diputadas de Chile, https://www.camara.cl/transparencia/datosAbiertos.aspx

