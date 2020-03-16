<template>
    <Page>
        <ActionBar title="COVID-19 Andamento italiano" class="action-bar" />

        <TabView height="100%" androidTabsPosition="top" :selectedIndex="0" @selectedIndexChange="indexChange">
            <TabViewItem title="Nazionale">
                <ScrollView>
                    <TabView height="100%" androidTabsPosition="top">
                        <TabViewItem title="Overview">
                            <FlexboxLayout justifyContent="space-between" flexDirection="column" class="home-panel contenitore">
                                <template v-if="lastEntry !== null">
                                    <HtmlView :html="`<strong>Totale Positivi:</strong> ${lastEntry.totale_attualmente_positivi} (${incremento(lastEntry.totale_attualmente_positivi, penultimateEntry.totale_attualmente_positivi)})`" />
                                    <HtmlView :html="`<strong>Tamponi:</strong> ${lastEntry.tamponi} (${incremento(lastEntry.tamponi, penultimateEntry.tamponi)})`" />
                                    <HtmlView class="guariti" :html="`<strong>Guariti:</strong> ${lastEntry.dimessi_guariti} (${incremento(lastEntry.dimessi_guariti, penultimateEntry.dimessi_guariti)})`" />
                                    <HtmlView class="deceduti" :html="`<strong>Deceduti:</strong> ${lastEntry.deceduti} (${incremento(lastEntry.deceduti, penultimateEntry.deceduti)})`" />

                                    <RadPieChart allowAnimation="true" row="0" height="200">
                                        <PieSeries v-tkPieSeries
                                                   selectionMode="DataPoint"
                                                   expandRadius="0.4"
                                                   outerRadiusFactor="0.7"
                                                   valueProperty="Amount"
                                                   legendLabel="Indice"
                                                   :items="[
                                               { Indice: 'Ricoverati con sintomi', Amount: lastEntry.ricoverati_con_sintomi },
                                               { Indice: 'Terapia intensiva', Amount: lastEntry.terapia_intensiva },
                                               { Indice: 'Isolamento Domiciliare', Amount: lastEntry.isolamento_domiciliare },
                                               ]" />

                                        <RadLegendView v-tkPieLegend position="Right" title="Infetti" offsetOrigin="TopRight" width="110" enableSelection="true"></RadLegendView>
                                    </RadPieChart>
                                </template>

                                <HtmlView :html="`<em>Aggiornamento: ${lastUpdate}</em>`" />

                                <HtmlView alignSelf="flex-end" class="callout callout-success" v-if="piccoSuperato" :html="`<strong>Forse abbiamo superato il picco!</strong> Questo non significa che è tutto finito, ma finalmente i nostri sforzi stanno iniziando a dare i loro risultati... grazie!`" />
                                <HtmlView alignSelf="flex-end" class="callout callout-danger" v-else :html="`<strong>Picco non ancora superato.</strong> Dobbiamo impegnarci per migliorare la situazione, fai anche tu la tua parte tenendoti informato e limitando i contatti. Per non sentirti solo, condividi la tua esperienza usando l'hashtag <em>#IORESTOACASA</em>`" />
                            </FlexboxLayout >
                        </TabViewItem>

                        <TabViewItem title="Dettaglio">
                            <ScrollView>
                                <StackLayout class="home-panel">
                                    <template v-if="lastEntry !== null">
                                        <HtmlView class="chart-title" :html="`Dove sono i malati?`" />
                                        <RadPieChart allowAnimation="true" row="0" height="200">
                                            <PieSeries v-tkPieSeries
                                                       selectionMode="DataPoint"
                                                       expandRadius="0.4"
                                                       outerRadiusFactor="0.7"
                                                       valueProperty="Amount"
                                                       legendLabel="Indice"
                                                       :items="[
                                               { Indice: 'Ricoverati con sintomi', Amount: lastEntry.ricoverati_con_sintomi },
                                               { Indice: 'Terapia intensiva', Amount: lastEntry.terapia_intensiva },
                                               { Indice: 'Isolamento Domiciliare', Amount: lastEntry.isolamento_domiciliare },
                                               ]" />

                                            <RadLegendView v-tkPieLegend position="Right" offsetOrigin="TopRight" width="110" enableSelection="true"></RadLegendView>
                                        </RadPieChart>
                                    </template>

                                    <template v-for="(chart, chart_key) in charts">
                                        <HtmlView class="chart-title" :html="chart.title" />
                                        <HtmlView v-if="typeof chart.explanation !== 'undefined'" class="callout callout-explanation" :html="chart.explanation" />
                                        <RadCartesianChart seriesSelectionMode="Single" :height="chart.height" style="font-size: 8;">
                                            <LineSeries
                                                    v-for="(serie, serie_key) in chart.serie"
                                                    :key="`nazionale.${chart_key}.${serie_key}`"
                                                    :seriesName="serie.name" :legendTitle="serie.title"
                                                    v-tkCartesianSeries
                                                    :items="entriesSorted"
                                                    selectionMode="Single"
                                                    categoryProperty="timestamp"
                                                    :valueProperty="serie.valore"/>
                                            <DateTimeContinuousAxis v-tkCartesianHorizontalAxis
                                                                    labelFitMode="Rotate" labelRotationAngle="-45"
                                                                    allowZoom="true" allowPan="true" majorStep="1" majorStepUnit="Week" />
                                            <LinearAxis v-tkCartesianVerticalAxis allowZoom="true" allowPan="true"
                                                        :labelFormat="typeof chart.y_label !== 'undefined' ? chart.y_label : '%.0f'"
                                                        minimum="0" />
                                            <RadLegendView v-tkCartesianLegend position="top" enableSelection="true"></RadLegendView>
                                        </RadCartesianChart>
                                    </template>
                                </StackLayout>
                            </ScrollView>
                        </TabViewItem>

                        <TabViewItem title="Trend">
                            <ScrollView>
                                <StackLayout class="home-panel">
                                    <HtmlView class="callout callout-info" :html="`Questi grafici mostrano <strong>variazioni percentuali</strong> rispetto al giorno precedente, per cui possono avere forti oscillazioni soprattutto quando i valori sono piccoli. Se, ad esempio, i dimessi guariti sono 10 il primo giorno, 20 il secondo e 15 il terzo, avremo dei valori di 0, +100% e - 25%.`" />
                                    <template v-for="(chart, chart_key) in trendCharts">
                                        <HtmlView class="chart-title" :html="chart.title" />
                                        <HtmlView v-if="typeof chart.explanation !== 'undefined'" class="callout callout-explanation" :html="chart.explanation" />
                                        <RadCartesianChart seriesSelectionMode="Single" :height="chart.height" style="font-size: 8;">
                                            <LineSeries
                                                    v-for="(serie, serie_key) in chart.serie"
                                                    :key="`nazionale.${chart_key}.${serie_key}`"
                                                    :seriesName="serie.name" :legendTitle="serie.title"
                                                    v-tkCartesianSeries
                                                    :items="entriesSorted"
                                                    selectionMode="Single"
                                                    categoryProperty="timestamp"
                                                    :valueProperty="serie.valore"/>
                                            <DateTimeContinuousAxis v-tkCartesianHorizontalAxis
                                                                    labelFitMode="Rotate" labelRotationAngle="-45"
                                                                    allowZoom="true" allowPan="true" majorStep="1" majorStepUnit="Week" />
                                            <LinearAxis v-tkCartesianVerticalAxis allowZoom="true" allowPan="true"
                                                        :labelFormat="typeof chart.y_label !== 'undefined' ? chart.y_label : '%.0f'"
                                                        maximum="100" minimum="-100" />
                                            <RadLegendView v-tkCartesianLegend position="top" enableSelection="true"></RadLegendView>
                                        </RadCartesianChart>
                                    </template>
                                </StackLayout>
                            </ScrollView>
                        </TabViewItem>

                    </TabView>
                </ScrollView>
            </TabViewItem>

            <TabViewItem title="Regionale">
                <ScrollView>
                    <TabView height="100%" androidTabsPosition="top">
                        <TabViewItem title="Overview">
                            <FlexboxLayout justifyContent="space-between" flexDirection="column" class="home-panel contenitore">
                                <ListPicker alignSelf="flex-start" :items="regioniSelezionabili"
                                            @selectedIndexChange="selectRegione" width="100%" />
                                <template v-if="lastEntry !== null">
                                    <HtmlView :html="`<strong>Totale Positivi:</strong> ${lastEntry.totale_attualmente_positivi} (${incremento(lastEntry.totale_attualmente_positivi, penultimateEntry.totale_attualmente_positivi)})`" />
                                    <HtmlView :html="`<strong>Tamponi:</strong> ${lastEntry.tamponi} (${incremento(lastEntry.tamponi, penultimateEntry.tamponi)})`" />
                                    <HtmlView class="guariti" :html="`<strong>Guariti:</strong> ${lastEntry.dimessi_guariti} (${incremento(lastEntry.dimessi_guariti, penultimateEntry.dimessi_guariti)})`" />
                                    <HtmlView class="deceduti" :html="`<strong>Deceduti:</strong> ${lastEntry.deceduti} (${incremento(lastEntry.deceduti, penultimateEntry.deceduti)})`" />

                                    <HtmlView :html="`<em>Aggiornamento: ${lastUpdate}</em>`" />

                                    <HtmlView alignSelf="flex-end" class="callout callout-success" v-if="piccoSuperato" :html="`<strong>Forse abbiamo superato il picco!</strong> Questo non significa che è tutto finito, ma finalmente i nostri sforzi stanno iniziando a dare i loro risultati... grazie!`" />
                                    <HtmlView alignSelf="flex-end" class="callout callout-danger" v-else :html="`<strong>Picco non ancora superato.</strong> Dobbiamo impegnarci per migliorare la situazione, fai anche tu la tua parte tenendoti informato e limitando i contatti. Per non sentirti solo, condividi la tua esperienza usando l'hashtag <em>#IORESTOACASA</em>`" />
                                </template>
                            </FlexboxLayout >
                        </TabViewItem>

                        <TabViewItem title="Dettaglio">
                            <ScrollView>
                                <StackLayout class="home-panel">
                                    <template v-if="lastEntry !== null">
                                        <HtmlView class="chart-title" :html="`Dove sono i malati?`" />
                                        <RadPieChart allowAnimation="true" row="0" height="200">
                                            <PieSeries v-tkPieSeries
                                                       selectionMode="DataPoint"
                                                       expandRadius="0.4"
                                                       outerRadiusFactor="0.7"
                                                       valueProperty="Amount"
                                                       legendLabel="Indice"
                                                       :items="[
                                               { Indice: 'Ricoverati con sintomi', Amount: lastEntry.ricoverati_con_sintomi },
                                               { Indice: 'Terapia intensiva', Amount: lastEntry.terapia_intensiva },
                                               { Indice: 'Isolamento Domiciliare', Amount: lastEntry.isolamento_domiciliare },
                                               ]" />

                                            <RadLegendView v-tkPieLegend position="Right" offsetOrigin="TopRight" width="110" enableSelection="true"></RadLegendView>
                                        </RadPieChart>
                                    </template>

                                    <template v-for="(chart, chart_key) in charts">
                                        <HtmlView class="chart-title" :html="chart.title" />
                                        <HtmlView v-if="typeof chart.explanation !== 'undefined'" class="callout callout-explanation" :html="chart.explanation" />
                                        <RadCartesianChart seriesSelectionMode="Single" :height="chart.height" style="font-size: 8;">
                                            <LineSeries
                                                    v-for="(serie, serie_key) in chart.serie"
                                                    :key="`nazionale.${chart_key}.${serie_key}`"
                                                    :seriesName="serie.name" :legendTitle="serie.title"
                                                    v-tkCartesianSeries
                                                    :items="entriesSorted"
                                                    selectionMode="Single"
                                                    categoryProperty="timestamp"
                                                    :valueProperty="serie.valore"/>
                                            <DateTimeContinuousAxis v-tkCartesianHorizontalAxis
                                                                    labelFitMode="Rotate" labelRotationAngle="-45"
                                                                    allowZoom="true" allowPan="true" majorStep="1" majorStepUnit="Week" />
                                            <LinearAxis v-tkCartesianVerticalAxis allowZoom="true" allowPan="true"
                                                        :labelFormat="typeof chart.y_label !== 'undefined' ? chart.y_label : '%.0f'"
                                                        minimum="0" />
                                            <RadLegendView v-tkCartesianLegend position="top" enableSelection="true"></RadLegendView>
                                        </RadCartesianChart>
                                    </template>
                                </StackLayout>
                            </ScrollView>
                        </TabViewItem>

                        <TabViewItem title="Trend">
                            <ScrollView>
                                <StackLayout class="home-panel">
                                    <HtmlView class="callout callout-info" :html="`Questi grafici mostrano <strong>variazioni percentuali</strong> rispetto al giorno precedente, per cui possono avere forti oscillazioni soprattutto quando i valori sono piccoli. Se, ad esempio, i dimessi guariti sono 10 il primo giorno, 20 il secondo e 15 il terzo, avremo dei valori di 0, +100% e - 25%.`" />
                                    <template v-for="(chart, chart_key) in trendCharts">
                                        <HtmlView class="chart-title" :html="chart.title" />
                                        <HtmlView v-if="typeof chart.explanation !== 'undefined'" class="callout callout-explanation" :html="chart.explanation" />
                                        <RadCartesianChart seriesSelectionMode="Single" :height="chart.height" style="font-size: 8;">
                                            <LineSeries
                                                    v-for="(serie, serie_key) in chart.serie"
                                                    :key="`nazionale.${chart_key}.${serie_key}`"
                                                    :seriesName="serie.name" :legendTitle="serie.title"
                                                    v-tkCartesianSeries
                                                    :items="entriesSorted"
                                                    selectionMode="Single"
                                                    categoryProperty="timestamp"
                                                    :valueProperty="serie.valore"/>
                                            <DateTimeContinuousAxis v-tkCartesianHorizontalAxis
                                                                    labelFitMode="Rotate" labelRotationAngle="-45"
                                                                    allowZoom="true" allowPan="true" majorStep="1" majorStepUnit="Week" />
                                            <LinearAxis v-tkCartesianVerticalAxis allowZoom="true" allowPan="true"
                                                        :labelFormat="typeof chart.y_label !== 'undefined' ? chart.y_label : '%.0f'"
                                                        maximum="100" minimum="-100" />
                                            <RadLegendView v-tkCartesianLegend position="top" enableSelection="true"></RadLegendView>
                                        </RadCartesianChart>
                                    </template>
                                </StackLayout>
                            </ScrollView>
                        </TabViewItem>
                    </TabView>
                </ScrollView>
            </TabViewItem>

            <TabViewItem title="Provinciale">
                <ScrollView>
                    <TabView height="100%" androidTabsPosition="top">
                        <TabViewItem title="Overview">
                            <FlexboxLayout justifyContent="space-between" flexDirection="column" class="home-panel contenitore">
                                <ListPicker alignSelf="flex-start" :items="provinceSelezionabili"
                                            @selectedIndexChange="selectProvincia" width="100%" />
                                <template v-if="lastEntry !== null">
                                    <HtmlView :html="`<strong>Totale Casi:</strong> ${lastEntry.totale_casi} (${incremento(lastEntry.totale_casi, penultimateEntry.totale_casi)})`" />
                                    <HtmlView :html="`<em>Aggiornamento: ${lastUpdate}</em>`" />
                                </template>

                                <HtmlView alignSelf="flex-end" class="callout callout-info" :html="`<strong>Nota sui dati:</strong> i dati provinciali non offrono spazio a elaborazioni in quanto viene fornito solo il numero totale dei casi.`" />
                            </FlexboxLayout >
                        </TabViewItem>

                        <TabViewItem title="Dettaglio">
                            <ScrollView>
                                <StackLayout class="home-panel">
                                    <template v-for="(chart, chart_key) in chartsProvince">
                                        <HtmlView class="chart-title" :html="chart.title" />
                                        <HtmlView v-if="typeof chart.explanation !== 'undefined'" class="callout callout-explanation" :html="chart.explanation" />
                                        <RadCartesianChart seriesSelectionMode="Single" :height="chart.height" style="font-size: 8;">
                                            <LineSeries
                                                    v-for="(serie, serie_key) in chart.serie"
                                                    :key="`nazionale.${chart_key}.${serie_key}`"
                                                    :seriesName="serie.name" :legendTitle="serie.title"
                                                    v-tkCartesianSeries
                                                    :items="entriesSorted"
                                                    selectionMode="Single"
                                                    categoryProperty="timestamp"
                                                    :valueProperty="serie.valore"/>
                                            <DateTimeContinuousAxis v-tkCartesianHorizontalAxis
                                                                    labelFitMode="Rotate" labelRotationAngle="-45"
                                                                    allowZoom="true" allowPan="true" majorStep="1" majorStepUnit="Week" />
                                            <LinearAxis v-tkCartesianVerticalAxis allowZoom="true" allowPan="true"
                                                        :labelFormat="typeof chart.y_label !== 'undefined' ? chart.y_label : '%.0f'"
                                                        minimum="0" />
                                            <RadLegendView v-tkCartesianLegend position="top" enableSelection="true"></RadLegendView>
                                        </RadCartesianChart>
                                    </template>
                                </StackLayout>
                            </ScrollView>
                        </TabViewItem>

                        <TabViewItem title="Trend">
                            <ScrollView>
                                <StackLayout class="home-panel">
                                    <HtmlView class="callout callout-info" :html="`Questi grafici mostrano <strong>variazioni percentuali</strong> rispetto al giorno precedente, per cui possono avere forti oscillazioni soprattutto quando i valori sono piccoli. Se, ad esempio, i nuovi casi sono 10 il primo giorno, 20 il secondo e 15 il terzo, avremo dei valori di 0, +100% e - 25%.`" />
                                    <template v-for="(chart, chart_key) in trendChartsProvince">
                                        <HtmlView class="chart-title" :html="chart.title" />
                                        <HtmlView v-if="typeof chart.explanation !== 'undefined'" class="callout callout-explanation" :html="chart.explanation" />
                                        <RadCartesianChart seriesSelectionMode="Single" :height="chart.height" style="font-size: 8;">
                                            <LineSeries
                                                    v-for="(serie, serie_key) in chart.serie"
                                                    :key="`nazionale.${chart_key}.${serie_key}`"
                                                    :seriesName="serie.name" :legendTitle="serie.title"
                                                    v-tkCartesianSeries
                                                    :items="entriesSorted"
                                                    selectionMode="Single"
                                                    categoryProperty="timestamp"
                                                    :valueProperty="serie.valore"/>
                                            <DateTimeContinuousAxis v-tkCartesianHorizontalAxis
                                                                    labelFitMode="Rotate" labelRotationAngle="-45"
                                                                    allowZoom="true" allowPan="true" majorStep="1" majorStepUnit="Week" />
                                            <LinearAxis v-tkCartesianVerticalAxis allowZoom="true" allowPan="true"
                                                        :labelFormat="typeof chart.y_label !== 'undefined' ? chart.y_label : '%.0f'"
                                                        maximum="100" minimum="-100" />
                                            <RadLegendView v-tkCartesianLegend position="top" enableSelection="true"></RadLegendView>
                                        </RadCartesianChart>
                                    </template>
                                </StackLayout>
                            </ScrollView>
                        </TabViewItem>
                    </TabView>
                </ScrollView>
            </TabViewItem>

            <TabViewItem title="Credits">
                <FlexboxLayout justifyContent="space-between" flexDirection="column" class="home-panel contenitore" width="100%" height="100%">
                    <HtmlView :html="`Questa applicazione è un progetto open source sviluppato da Improoving.me srl allo scopo di sensibilizzare la popolazione nei confronti della diffusione del virus SARS-COV-2.`" />
                    <HtmlView :html="`Tutti i dati utilizzati sono forniti dalla Protezione Civile al link https://github.com/pcm-dpc/COVID-19`" />
                    <HtmlView :html="`Se sei uno sviluppatore e desideri migliorare questa applicazione, trovi il file sorgente al link https://github.com/improoving-me/covid-19`" />
                    <Image src="https://improoving.me/img/logo.png"
                           width="50%" margin="5px 0 20px" />
                    <HtmlView alignSelf="flex-end" alignItems="center" class="callout callout-success" :html="`<p><strong>Andrà tutto bene</strong> 🌈</p>`" width="100%" />
                </FlexboxLayout>
            </TabViewItem>

        </TabView>
    </Page>
</template>

<script>
    import Vue from "nativescript-vue";
    import RadCartesianChart from "nativescript-ui-chart/vue";
    import Moment from "moment";

    Vue.use(RadCartesianChart);
    Vue.use(Moment);

    export default {
    	props: {
    		charts: {
    			default(){
    				return [
                        {
                        	title: 'Quanti sono stati colpiti dal virus?',
	                        height: 300,
                            explanation: 'Da questo grafico possiamo vedere come evolve il numero di positivi e degli ospedalizzati nel tempo. Questo grafico è cumulativo, ciò significa che rappresenta il totale delle persone testate positive per il COVID-19.',
                            serie: [
                            	{
                            		title: 'Totale infetti',
                                    name: 'Totale infetti',
		                            valore: 'totale_casi'
                                },
	                            {
		                            title: 'Totale Ospedalizzati',
		                            name: 'Totale Ospedalizzati',
		                            valore: 'totale_ospedalizzati',
	                            },
                            ]
                        },
					    {
						    title: 'A che velocità si diffonde il virus?',
						    height: 300,
						    explanation: 'Questo grafico mostra quante persone, giornalmente, risultano positive al COVID-19. lo scopo di questa guerra, il motivo per cui ci chiedono di restare a casa, è quello di combattere per abbassare questo valore al minimo.',
						    serie: [
							    {
								    title: 'Nuovi Positivi',
								    name: 'Nuovi Positivi',
								    valore: 'nuovi_attualmente_positivi',
							    },
						    ]
					    },
					    {
						    title: 'La gente guarisce?',
						    height: 300,
						    explanation: 'In questa tabella possiamo vedere l`andamento dei dimessi guariti e dei deceduti. Questo grafico è cumulativo.',
						    serie: [
							    {
								    title: 'Guariti',
								    name: 'Guariti',
								    valore: 'dimessi_guariti',
							    },
							    {
								    title: 'Deceduti',
								    name: 'Deceduti',
								    valore: 'deceduti',
							    },
						    ]
					    },
					    {
						    title: 'Come vengono curati i malati?',
						    height: 300,
						    explanation: 'In base alla severità della malattia i malati vengono messi in isolamento domestico, ricoverati in ospedale o, nei casi peggiori, ricoverati in terapia intensiva. Da questo grafico possiamo vedere come il nostro Sistema Sanitario sta distribuendo i malati nel tempo.',
						    serie: [
							    {
								    title: 'Ricoverati con sintomi',
								    name: 'Ricoverati con sintomi',
								    valore: 'ricoverati_con_sintomi',
							    },
							    {
								    title: 'Terapia intensiva',
								    name: 'Terapia intensiva',
								    valore: 'terapia_intensiva',
							    },
							    {
								    title: 'Isolamento domiciliare',
								    name: 'Isolamento domiciliare',
								    valore: 'isolamento_domiciliare',
							    }
						    ]
					    },
					    {
						    title: 'Quanti test stiamo facendo?',
						    height: 300,
						    explanation: 'Questo grafico mostra il totale dei tamponi effettuati dall`inizio dell`emergenza. Questo grafico è cumulativo.',
						    serie: [
							    {
								    title: 'Tamponi effettuati',
								    name: 'Tamponi effettuati',
								    valore: 'tamponi',
							    }
						    ]
					    },
                    ];
                }
            },
    		trendCharts: {
    			default(){
    				return [
                        {
                        	title: 'A che velocità si diffonde il contagio?',
	                        height: 300,
                            y_label: '%.0f pp',
                            explanation: 'Da questo grafico possiamo capire se il contagio sta rallentando; il valore ottimale è sotto lo 0, a significare che i contagiati giornalieri sono minori del giorno precedente.',
                            serie: [
	                            {
		                            title: 'Totale infetti',
		                            name: 'Totale infetti',
		                            valore: 'incremento_totale_casi',
	                            },
	                            {
		                            title: 'Nuovi positivi',
		                            name: 'Nuovi positivi',
		                            valore: 'incremento_nuovi_attualmente_positivi',
	                            },
                            ]
                        },
                        {
                        	title: 'A che velocità crescono i malati?',
	                        height: 300,
                            y_label: '%.0f pp',
                            explanation: 'Da questo grafico possiamo capire la velocità con la quale i nostri ospedali si stanno riempendo. Anche per questo grafico i valori ottimali sono sotto lo 0.',
                            serie: [
	                            {
		                            title: 'Ricoverati con sintomi',
		                            name: 'Ricoverati con sintomi',
		                            valore: 'incremento_ricoverati_con_sintomi',
	                            },
	                            {
		                            title: 'Terapia intensiva',
		                            name: 'Terapia intensiva',
		                            valore: 'incremento_terapia_intensiva',
	                            },
	                            {
		                            title: 'Isolamento domiciliare',
		                            name: 'Isolamento domiciliare',
		                            valore: 'incremento_isolamento_domiciliare',
	                            }
                            ]
                        },
                        {
                        	title: 'Come procedono le cure?',
	                        height: 300,
                            y_label: '%.0f pp',
                            explanation: 'Questo grafico, idealmente, dovrebbe mostrare due trend di segno opposto: i deceduti in calo ed i guariti in crescita, a significare che la nostra efficacia nel gestire l`emergenza sta migliorando. Questo grafico è fortemente soggetto a oscillazioni.',
                            serie: [
	                            {
		                            title: 'Guariti',
		                            name: 'Guariti',
		                            valore: 'incremento_dimessi_guariti',
	                            },
	                            {
		                            title: 'Deceduti',
		                            name: 'Deceduti',
		                            valore: 'incremento_deceduti',
	                            }
                            ]
                        },
                        {
                        	title: 'A che velocità crescono i tamponi?',
	                        height: 300,
                            y_label: '%.0f pp',
                            explanation: 'Questo grafico mostra la velocità con la quale cresce il numero di tamponi effettuati giornalmente.',
                            serie: [
	                            {
		                            title: 'Tamponi effettuati',
		                            name: 'Tamponi effettuati',
		                            valore: 'incremento_tamponi',
	                            }
                            ]
                        },
                    ];
                }
            },
            chartsProvince:{
                default(){
		            return [
			            {
				            title: 'Quanti sono stati colpiti dal virus?',
				            height: 300,
				            explanation: 'Da questo grafico possiamo vedere come evolve il numero di positivi e degli ospedalizzati nel tempo. Questo grafico è cumulativo, ciò significa che rappresenta il totale delle persone testate positive per il COVID-19.',
				            serie: [
					            {
						            title: 'Totale Infetti',
						            name: 'Totale Infetti',
						            valore: 'totale_casi'
					            }
				            ],
			            },
			            {
				            title: 'A che velocità si diffonde il virus?',
				            height: 300,
				            explanation: 'Questo grafico mostra quante persone, giornalmente, risultano positive al COVID-19. lo scopo di questa guerra, il motivo per cui ci chiedono di restare a casa, è quello di combattere per abbassare questo valore al minimo.',
				            serie: [
					            {
						            title: 'Nuovi Positivi',
						            name: 'Nuovi Positivi',
						            valore: 'nuovi_attualmente_positivi',
					            },
				            ]
			            },
		            ];
	            }
            },
            trendChartsProvince:{
                default(){
		            return [
			            {
				            title: 'A che velocità si diffonde il contagio?',
				            height: 300,
				            y_label: '%.0f pp',
				            explanation: 'Da questo grafico possiamo capire se il contagio sta rallentando; il valore ottimale è sotto lo 0, a significare che i contagiati giornalieri sono minori del giorno precedente.',
				            serie: [
					            {
						            title: 'Totale infetti',
						            name: 'Totale infetti',
						            valore: 'incremento_totale_casi',
					            }
				            ]
			            },
		            ];
	            }
            },
            parametri: {
    			default(){
    				return ['ricoverati_con_sintomi', 'terapia_intensiva', 'totale_ospedalizzati', 'isolamento_domiciliare', 'totale_attualmente_positivi', 'nuovi_attualmente_positivi', 'dimessi_guariti', 'deceduti', 'totale_casi', 'tamponi'];
                }
            },
            parametriProvince: {
    			default(){
    				return ['totale_casi'];
                }
            },
        },
        data() {
            return {
                dati_nazionali: [],
	            dati_regionali: [],
	            dati_regionali_elaborati: {},
                dati_provinciali: [],
                dati_provinciali_elaborati: {},
                url_nazione: "https://raw.githubusercontent.com/pcm-dpc/COVID-19/master/dati-json/dpc-covid19-ita-andamento-nazionale.json",
                url_regioni: "https://raw.githubusercontent.com/pcm-dpc/COVID-19/master/dati-json/dpc-covid19-ita-regioni.json",
                url_province: "https://raw.githubusercontent.com/pcm-dpc/COVID-19/master/dati-json/dpc-covid19-ita-province.json",
                visualizzazione: 'dati_nazionali',
	            regione_selezionata: null,
                provincia_selezionata: null,
                timeout_regione: false,
                timeout_provincia: false
            };
        },
        computed: {
        	lastUpdate(){
        		return this.lastEntry !== null ?
                    Moment(this.lastEntry.data).format('DD/MM/YYYY') : 'Dati non scaricati, ricaricare.';
            },
            lastEntry() {
	            return this.entriesSorted.length ?
		            this.entriesSorted[this.entriesSorted.length - 1] : null;
            },
            penultimateEntry() {
	            return this.entriesSorted.length ?
		            this.entriesSorted[this.entriesSorted.length - 2] : null;
            },
            entriesSorted(){
        		let _this = this,
                    consider = _this.visualizzazione === 'dati_provinciali' ? 'parametriProvince' : 'parametri';
	            return _this.entries.length ?
		            _this.entries.sort(function(a, b){
		            	return Moment(a.data).isAfter(b.data) ? 1 : -1;
                    }).map(function(el, key){
			            el.data_elaborata = Moment(el.data).format('DD/MM/YYYY');
			            el.timestamp = Moment(el.data);
			            let previousDateEntry = _this.previousDateEntry(_this.entries, el);
			            _this[consider].forEach(function(parametro){
				            let new_parametro = ['incremento', parametro].join('_');
				            el[new_parametro] = typeof previousDateEntry !== 'undefined' ?
					            _this.variazionePercentuale(el[parametro], previousDateEntry[parametro]) : 0;
			            });

			            if(consider === 'parametriProvince')
				            el.nuovi_attualmente_positivi = typeof previousDateEntry !== 'undefined' ?
                                (el.totale_casi - previousDateEntry.totale_casi) : el.totale_casi;

			            return el;
		            }) : [];
            },
            entries(){
        		return this.visualizzazione === 'dati_nazionali' ?
                    this.entriesNazionali
                    : (this.visualizzazione === 'dati_regionali' ? this.entriesRegionali
                        : (this.visualizzazione === 'dati_provinciali' ? this.entriesProvinciali : [] ));
            },
            entriesNazionali(){
        		return this[this.visualizzazione].length ? this[this.visualizzazione] : [];
            },
            entriesRegionali(){
        		let _this = this;
        		return _this.regioniSelezionabili.length && _this.regione_selezionata !== null && typeof _this.regioniSelezionabili[_this.regione_selezionata] !== 'undefined' ?
			        Object.entries(this.dati_regionali_elaborati).find(function(el){
                    	return el[1].denominazione_regione === _this.regioniSelezionabili[_this.regione_selezionata];
                    })[1].entries : [];
            },
            entriesProvinciali(){
        		let _this = this;
        		return _this.provinceSelezionabili.length && _this.provincia_selezionata !== null && typeof _this.provinceSelezionabili[_this.provincia_selezionata] !== 'undefined' ?
			        Object.entries(this.dati_provinciali_elaborati).find(function(el){
                    	return [el[1].denominazione_regione,el[1].denominazione_provincia].join(' - ') === _this.provinceSelezionabili[_this.provincia_selezionata];
                    })[1].entries : [];
            },
	        piccoSuperato(){
        		return this.lastEntry !== null && this.entriesSorted.length > 3 && Moment(new Date()).isAfter('2020-03-20') ?
                    (this.lastEntry.nuovi_attualmente_positivi < this.penultimateEntry.nuovi_attualmente_positivi
                            && this.penultimateEntry.nuovi_attualmente_positivi < this.entriesSorted[this.entriesSorted.length - 3].nuovi_attualmente_positivi
                    ) : false
            },
            regioniSelezionabili(){
	            return this.visualizzazione === 'dati_regionali' && Object.entries(this.dati_regionali_elaborati).length ?
		            Object.entries(this.dati_regionali_elaborati).map(function(el){
			            return el[1].denominazione_regione;
		            }).sort(function(a, b){
		            	return a > b ? 1 : -1;
                    }) : [];
            },
	        provinceSelezionabili(){
		        return this.visualizzazione === 'dati_provinciali' && Object.entries(this.dati_provinciali_elaborati).length ?
			        Object.entries(this.dati_provinciali_elaborati).map(function(el){
				        return [el[1].denominazione_regione,el[1].denominazione_provincia].join(' - ');
			        }).sort(function(a, b){
				        return a > b ? 1 : -1;
			        }) : [];
	        }
        },
        created() {
    		this.fetchNazionali();
        },
        watch: {
	        entriesRegionali(newValue){},
	        regioniSelezionabili(newValue){},
	        provinceSelezionabili(newValue){}
        },
        methods: {
    		indexChange(args){
    			let _this = this,
                    value = parseInt(args.value);

    			if([0,1,2].includes(value)){
    				let visualizzazione = 'dati_nazionali';
    				if(value === 1)
    					visualizzazione = 'dati_regionali';
    				else if(value === 2)
					    visualizzazione = 'dati_provinciali';

				    if(_this[visualizzazione].length === 0){
					    if(visualizzazione === 'dati_nazionali')
						    _this.fetchNazionali().then(function(){
							    _this.visualizzazione = visualizzazione;
						    });
					    else if(visualizzazione === 'dati_regionali')
						    _this.fetchRegionali().then(function(){
							    _this.visualizzazione = visualizzazione;
						    });
					    else if(visualizzazione === 'dati_provinciali')
						    _this.fetchProvinciali().then(function(){
							    _this.visualizzazione = visualizzazione;
						    });
				    }
				    else
					    _this.visualizzazione = visualizzazione;
                }
            },
        	incremento(a, b){
    			let variazione = this.variazionePercentuale(a, b);
        		return [a > b ? '+':'', isNaN(variazione) ? '0.00' : this.variazionePercentuale(a, b).toFixed(2), '%'].join('');
            },
            variazionePercentuale(a, b){
    			try{
				    return (a-b)*100/b;
                }
                catch(e){
    				return 0;
                }
            },
	        fetchNazionali(){
		        let _this = this;
		        return fetch(_this.url_nazione)
		        .then(response => response.json())
		        .then(function(r) {
			        _this.dati_nazionali = r;

			        console.log('database Nazionale scaricato');
		        });
	        },
	        fetchRegionali(){
		        let _this = this;
		        return fetch(_this.url_regioni)
		        .then(response => response.json())
		        .then(function(r) {
			        _this.dati_regionali = r;
			        console.log('database Regionale scaricato');
			        r.forEach(function(item, key) {
			        	if(item.lat !== 0){
					        if (typeof _this.dati_regionali_elaborati[item.codice_regione] === "undefined")
						        _this.dati_regionali_elaborati[item.codice_regione] = {
							        codice_regione: item.codice_regione,
							        denominazione_regione: item.denominazione_regione,
							        entries: []
						        };

					        _this.dati_regionali_elaborati[item.codice_regione].entries.push(item);
                        }
			        });
			        console.log('database Regionale elaborato');
			        // console.log(_this.dati_regionali_elaborati);
		        });
	        },
	        fetchProvinciali(){
		        let _this = this;
		        return fetch(_this.url_province)
		        .then(response => response.json())
		        .then(function(r) {
			        _this.dati_provinciali = r;
			        console.log('database Provinciale scaricato');
			        r.forEach(function(item, key) {
			        	if(item.lat !== 0){
					        if (typeof _this.dati_provinciali_elaborati[item.codice_provincia] === "undefined")
						        _this.dati_provinciali_elaborati[item.codice_provincia] = {
							        codice_regione: item.codice_regione,
							        denominazione_regione: item.denominazione_regione,
							        codice_provincia: item.codice_provincia,
							        denominazione_provincia: item.denominazione_provincia,
							        entries: []
						        };

					        _this.dati_provinciali_elaborati[item.codice_provincia].entries.push(item);
                        }
			        });
			        console.log('database Provinciale elaborato');
			        // console.log(_this.dati_provinciali_elaborati);
		        });
	        },
            previousDateEntry(entries, entry){
    			let date = Moment(entry.data).subtract(1, "day");

    			return entries.find(function(el){
    				return Moment(el.data).isSame(date, 'day');
                });
            },
            selectProvincia(args){
    			let _this = this;
	            clearTimeout(_this.timeout_provincia);

    			_this.timeout_provincia = setTimeout(function(){
    				_this.provincia_selezionata = args.value;
                }, 250);
            },
            selectRegione(args){
	            let _this = this;
	            clearTimeout(_this.timeout_regione);

	            _this.timeout_regione = setTimeout(function(){
		            _this.regione_selezionata = args.value;
	            }, 250);

            },
        }
    };
</script>

<style>
    .home-panel {
        vertical-align: center;
        font-size: 20;
        margin: 15;
    }

    .description-label {
        margin-bottom: 15;
        text-transform: rotate(45deg);
    }

    .contenitore {
        padding: 5px 15px;
        text-align: center;
    }

    .contenitore > * {
        margin: 0;
        padding: 0;
    }
    .contenitore h1,
    .contenitore h2,
    .contenitore h3,
    .contenitore h4,
    .contenitore h5,
    .contenitore h6 {
        margin: 0;
        padding: 0;
        font-size: 10px;
    }
    .contenitore p,
    .contenitore div {
        margin: 0 !important;
        padding: 0 !important;
    }
    .contenitore .guariti{
        color: green !important;
    }
    .contenitore .deceduti{
        color: red !important;
    }

    .callout{
        display: block;
        border-radius: 3rem;
        color: #000;
        font-size: 15px;
        margin: 0 -15px -5px -15px;
        padding: 15px 30px;
    }

    .callout-info{
        border-color: #1e9ff2!important;
        background-color: #b8e1fb;
    }

    .callout-success{
        border-color: #28d094!important;
        background-color: #acefd7;
    }

    .callout-danger{
        border-color: #e91e63!important;
        background-color: #f8b3ca;
    }

    .callout strong{
        font-size: 25px !important;
    }

    .chart-title{
        margin-top: 50px;
        font-weight: bold;
    }

    LinearAxis {
        label-text-color: black;
        line-hidden: false;
        line-color: black;
    }

    CategoricalAxis {
        label-text-color: black;
        line-hidden: false;
        line-color: black;
    }

    .text-center{
        text-align: center !important;
    }
</style>