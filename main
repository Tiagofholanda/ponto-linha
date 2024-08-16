import streamlit as st
import geopandas as gpd
from shapely.geometry import Point, LineString
import numpy as np
from scipy.spatial import KDTree
import os
import zipfile
import tempfile

def load_shapefile_from_zip(zip_file):
    # Criar um diretório temporário para descompactar os arquivos
    with tempfile.TemporaryDirectory() as tmpdirname:
        with zipfile.ZipFile(zip_file, 'r') as zip_ref:
            zip_ref.extractall(tmpdirname)
        
        # Encontrar o arquivo .shp dentro do diretório temporário
        shapefile_path = None
        for root, dirs, files in os.walk(tmpdirname):
            for file in files:
                if file.endswith(".shp"):
                    shapefile_path = os.path.join(root, file)
                    break

        if shapefile_path is None:
            st.error("Nenhum arquivo .shp encontrado no arquivo ZIP.")
            return None

        pontos = gpd.read_file(shapefile_path)
        
        # Verifica se o shapefile está em um sistema de coordenadas projetadas
        if pontos.crs.is_geographic:
            st.info("Reprojetando para UTM automaticamente.")
            pontos = pontos.to_crs(pontos.estimate_utm_crs())
        
        return pontos

def generate_lines(pontos):
    linhas = []
    pontos_array = np.array([(geom.x, geom.y) for geom in pontos.geometry])
    tree = KDTree(pontos_array)

    progress_bar = st.progress(0)

    for index, ponto in pontos.iterrows():
        dist, idx = tree.query((ponto.geometry.x, ponto.geometry.y), k=2)  # k=2 porque o primeiro é ele mesmo
        ponto_mais_proximo = pontos.iloc[idx[1]].geometry
        linha = LineString([ponto.geometry, ponto_mais_proximo])
        linhas.append(linha)

        progress_bar.progress((index + 1) / len(pontos))

    linhas_gdf = gpd.GeoDataFrame(geometry=linhas, crs=pontos.crs)
    return linhas_gdf

def save_shapefile(linhas_gdf, filename):
    linhas_gdf.to_file(filename)
    st.success(f"Shapefile salvo como '{filename}'")

def main():
    st.title("Gerador de Linhas entre Pontos")
    st.write("Este aplicativo permite que você carregue um shapefile de pontos em um arquivo ZIP, gere linhas conectando cada ponto ao seu vizinho mais próximo, e salve as linhas geradas como um novo shapefile.")

    uploaded_file = st.file_uploader("Carregar Shapefile ZIP", type=["zip"])
    if uploaded_file:
        pontos = load_shapefile_from_zip(uploaded_file)
        
        if pontos is not None:
            st.success("Shapefile carregado com sucesso!")
            
            # Renomear as colunas POINT_X e POINT_Y para longitude e latitude
            pontos = pontos.rename(columns={"POINT_X": "longitude", "POINT_Y": "latitude"})
            
            st.map(pontos[['latitude', 'longitude']])

            if st.button("Gerar Linhas"):
                linhas_gdf = generate_lines(pontos)
                st.success("Linhas geradas com sucesso!")
                
                # Extrair latitudes e longitudes das geometrias das linhas
                linhas_gdf['latitude'] = linhas_gdf.geometry.apply(lambda geom: geom.centroid.y)
                linhas_gdf['longitude'] = linhas_gdf.geometry.apply(lambda geom: geom.centroid.x)
                
                st.map(linhas_gdf[['latitude', 'longitude']])

                if st.button("Salvar Shapefile"):
                    save_filename = st.text_input("Nome do arquivo para salvar", "linhas_geradas.shp")
                    if save_filename:
                        save_shapefile(linhas_gdf, save_filename)

if __name__ == "__main__":
    main()
