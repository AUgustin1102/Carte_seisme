import pandas as pd
import folium
from geopy.distance import geodesic
from tqdm import tqdm

# 📂 Chargement des données
earthquakes_path = "/content/Seisme-magnitude.csv"
cities_path = "/content/worldcities.csv"

df_quakes = pd.read_csv(earthquakes_path)
df_cities = pd.read_csv(cities_path, sep=";", encoding="utf-8")

# 🛠️ Correction des latitudes/longitudes
df_cities["lat"] = df_cities["lat"].astype(str).str.replace(",", ".").astype(float)
df_cities["lng"] = df_cities["lng"].astype(str).str.replace(",", ".").astype(float)

# 🔎 Filtrer les villes avec +200 000 habitants
df_cities = df_cities[df_cities["population"] > 200000]

# 🔎 Filtrer les séismes avec magnitude ≥ 6.0
df_quakes = df_quakes[df_quakes['mag'] >= 6.0]

# 🎯 Fonction pour trouver les villes affectées dans un rayon de 100 km
def find_affected_cities(earthquake, cities, radius_km=100):
    affected = []
    quake_location = (earthquake['latitude'], earthquake['longitude'])
    
    for _, city in cities.iterrows():
        city_location = (city['lat'], city['lng'])
        distance = geodesic(quake_location, city_location).km
        
        if distance <= radius_km:
            affected.append((city['city_ascii'], city['country'], city_location, distance, city["population"]))
    
    return affected

# 📌 Trouver les séismes ayant des villes affectées
filtered_quakes = []
affected_cities = []

for _, quake in tqdm(df_quakes.iterrows(), total=df_quakes.shape[0]):
    cities_nearby = find_affected_cities(quake, df_cities)
    
    if cities_nearby:
        filtered_quakes.append(quake)
        
        for city in cities_nearby:
            affected_cities.append({
                "City": city[0],
                "Country": city[1],
                "Latitude": city[2][0],
                "Longitude": city[2][1],
                "Population": city[4],
                "Earthquake_Lat": quake['latitude'],
                "Earthquake_Lng": quake['longitude'],
                "Magnitude": quake['mag']
            })

df_filtered_quakes = pd.DataFrame(filtered_quakes)
df_affected_cities = pd.DataFrame(affected_cities)

# 🛠️ Garder uniquement le plus gros séisme par zone (par pas de 0.5°)
df_filtered_quakes["lat_rounded"] = df_filtered_quakes["latitude"].round(1)
df_filtered_quakes["lng_rounded"] = df_filtered_quakes["longitude"].round(1)
df_filtered_quakes = df_filtered_quakes.sort_values(by="mag", ascending=False).drop_duplicates(subset=["lat_rounded", "lng_rounded"])

# 🟢 Carte centrée sur la moyenne des séismes
m = folium.Map(location=[df_filtered_quakes['latitude'].mean(), df_filtered_quakes['longitude'].mean()], zoom_start=3)

# 🎨 Fonction pour les couleurs
def get_quake_color(mag):
    if 6.0 <= mag < 6.5:
        return "orange"
    elif 6.5 <= mag < 7.0:
        return "darkred"
    else:
        return "black"

def get_city_color(pop):
    if 200000 <= pop < 500000:
        return "lightblue"
    elif 500000 <= pop < 1000000:
        return "blue"
    else:
        return "purple"

# 🔴 Ajouter les séismes
for _, quake in df_filtered_quakes.iterrows():
    quake_color = get_quake_color(quake["mag"])
    
    folium.Circle(
        location=(quake['latitude'], quake['longitude']),
        radius=100000, 
        color=quake_color,
        fill=True,
        fill_color=quake_color,
        fill_opacity=0.3,
        popup=folium.Popup(f"""<b>Séisme</b><br>
                              <b>Magnitude:</b> {quake['mag']}<br>
                              <b>Latitude:</b> {quake['latitude']}<br>
                              <b>Longitude:</b> {quake['longitude']}""", max_width=300),
    ).add_to(m)

# 🔵 Ajouter les villes
for _, city in df_affected_cities.iterrows():
    city_color = get_city_color(city["Population"])
    
    folium.CircleMarker(
        location=(city["Latitude"], city["Longitude"]),
        radius=5,
        color=city_color,
        fill=True,
        fill_color=city_color,
        fill_opacity=0.9,
        popup=folium.Popup(f"""<b>Ville:</b> {city['City']}<br>
                            <b>Population:</b> {city['Population']}""", max_width=200),
    ).add_to(m)

# 🌁 Ajouter une légende
legend_html = """
<div style="
    position: fixed;
    bottom: 50px; left: 50px; width: 250px; height: 230px;
    background-color: white; z-index:9999; font-size:14px;
    border-radius: 10px; padding: 10px; box-shadow: 3px 3px 5px grey;
    ">
<b>Légende :</b><br>
<b>Séismes :</b><br>
<span style="color:orange;">&#11044;</span> Magnitude 6.0 - 6.5<br>
<span style="color:darkred;">&#11044;</span> Magnitude 6.5 - 7.0<br>
<span style="color:black;">&#11044;</span> Magnitude 7.0+<br><br>

<b>Villes :</b><br>
<span style="color:lightblue;">&#11044;</span> 200 000 - 500 000 habitants<br>
<span style="color:blue;">&#11044;</span> 500 000 - 1 million habitants<br>
<span style="color:purple;">&#11044;</span> 1 million + habitants<br>
</div>
"""

m.get_root().html.add_child(folium.Element(legend_html))

# 🔥 Sauvegarde de la carte
m.save("Carte_Seismes.html")
