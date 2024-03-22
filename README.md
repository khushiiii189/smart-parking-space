# smart-parking-spaceimport time
import time
import json
import folium
import pandas as pd
from selenium import webdriver
from folium.plugins import Geocoder
from http.server import BaseHTTPRequestHandler, HTTPServer
import json
import threading
import os  
import cv2
import numpy as np
from PIL import Image
from io import BytesIO
import csv

def find_popup_slice(html):
    '''
    Find the starting and edning index of popup function
    '''

    pattern = "function latLngPop(e)"

    # startinf index
    starting_index = html.find(pattern)
    #
    tmp_html = html[starting_index:]

    #
    found = 0
    index = 0
    opening_found = False
    while not opening_found or found > 0:
        if tmp_html[index] == "{":
            found += 1
            opening_found = True
        elif tmp_html[index] == "}":
            found -= 1

        index += 1

    # determine the edning index of popup function
    ending_index = starting_index + index

    return starting_index, ending_index

def find_variable_name(html, name_start):
    variable_pattern = "var "
    pattern = variable_pattern + name_start
    starting_index = html.find(pattern) + len(variable_pattern)
    tmp_html = html[starting_index:]
    ending_index = tmp_html.find(" =") + starting_index

    return html[starting_index:ending_index]

def custom_code(popup_variable_name, map_variable_name, folium_port):
    return '''
            // custom code
            function latLngPop(e) {
                %s
                    .setLatLng(e.latlng)
                    .setContent(`
                        lat: ${e.latlng.lat}, lng: ${e.latlng.lng}
                        <button onClick="
                            fetch('http://localhost:%s', {
                                method: 'POST',
                                mode: 'no-cors',
                                headers: {
                                    'Accept': 'application/json',
                                    'Content-Type': 'application/json'
                                },
                                body: JSON.stringify({
                                    latitude: ${e.latlng.lat},
                                    longitude: ${e.latlng.lng}
                                })
                            });

                            L.marker(
                                [${e.latlng.lat}, ${e.latlng.lng}],
                                {}
                            ).addTo(%s);


                        "> Analyze </button>
                        <button onClick="
                            fetch('http://localhost:%s', {
                                method: 'POST',
                                mode: 'no-cors',
                                headers: {
                                    'Accept': 'application/json',
                                    'Content-Type': 'application/json'
                                },
                                body: 'q'
                            });
                        "> Quit </button>
                    `)
                    .openOn(%s);
            }
            // end custom code
    ''' % (popup_variable_name, folium_port, map_variable_name, folium_port, map_variable_name)

def load_coords():
    try:
        with open('coords.json', 'r') as file:
            return json.load(file)
    except FileNotFoundError:
        return []
    
def create_folium_map(map_filepath, center_coord, folium_port):
        
    # create folium map
    data = pd.read_csv('new2.csv')
    
    df = pd.DataFrame(data)

    df = df.reset_index()

    # Preprocessing: Drop unwanted rows
    unwanted_categories = ['Underwear store', 'Auto repair shop', 'Motorcycle dealer', 'Parking garage',
                            'Fruit and vegetable store', 'Indian grocery store', 'Grocery store',
                            'General store', 'Produce market']
    
    df = df[~df['categoryName'].isin(unwanted_categories)]
    df = df[df['title'] != 'PUNIT OIL DEPOT']
    vmap = folium.Map(center_coord, zoom_start=17)

    # add popup
    folium.LatLngPopup().add_to(vmap)

    # store the map to a file
    vmap.save(map_filepath)
        
    basemaps = {
        'Google Maps': folium.TileLayer(tiles='https://mt1.google.com/vt/lyrs=m&x={x}&y={y}&z={z}', attr='Google',
                                        name='Google Maps', overlay=True, control=True),
        
        'Google Satellite': folium.TileLayer(tiles='https://mt1.google.com/vt/lyrs=s&x={x}&y={y}&z={z}', attr='Google',
                                            name='Google Satellite', overlay=True, control=True),
        
    }

    for layer in basemaps.values():
        layer.add_to(vmap)
    

    folium.LayerControl().add_to(vmap)
    html = None
    with open(map_filepath, 'r') as mapfile:
            html = mapfile.read()

    # find variable names
    map_variable_name = find_variable_name(html, "map_")
    popup_variable_name = find_variable_name(html, "lat_lng_popup_")

    # determine popup function indicies
    pstart, pend = find_popup_slice(html)

    # inject code
    with open(map_filepath, 'w') as mapfile:
        mapfile.write(
            html[:pstart] + \
            custom_code(popup_variable_name, map_variable_name, folium_port) + \
            html[pend:]
        )
 
    def add_marker(row, map_variable_name, folium_port):
          popup_content = f'''
          <div>
        <p>{row['title']}</p>
        <button id='ANALYZE' onClick="
            fetch('http://localhost:{folium_port}', {{
                method: 'POST',
                mode: 'no-cors',
                headers: {{
                    'Accept': 'application/json',
                    'Content-Type': 'application/json'
                }},
                body: JSON.stringify({{
                    latitude: {row['latitude']},
                    longitude:{row['longitude']},
                    capture_screenshot: true
                }})
            }});

            L.marker(
                [{row['latitude']}, {row['longitude']}],
                {{}}
            ).addTo({map_variable_name});
            
       
        "> Analyze </button>
        <button onClick="
            fetch('http://localhost:{folium_port}', {{
                method: 'POST',
                mode: 'no-cors',
                headers: {{
                    'Accept': 'application/json',
                    'Content-Type': 'application/json'
                }},
                body: 'q'
            }});
        " name='button'> Quit </button>
    </div>
    '''
          
          folium.Marker(
        location=[row['latitude'], row['longitude']],
         popup=folium.Popup(popup_content),
    ).add_to(vmap)
          
   


    df.apply(add_marker, axis=1, args=(map_variable_name,folium_port))
    
    Geocoder().add_to(vmap)
    vmap.save(map_filepath)

    

def open_folium_map( map_filepath):
    driver = None
    try:
        driver = webdriver.Chrome()
        driver.get(map_filepath)
    except Exception as ex:
        print(f"Driver failed to open/find url: {ex}")

    return driver


class FoliumServer(BaseHTTPRequestHandler):
    def _set_response(self):
        self.send_response(200)
        self.send_header('Content-type', 'text/html')
        self.end_headers()

    def do_POST(self):
        content_length = int(self.headers['Content-Length'])
        post_data = self.rfile.read(content_length)
        data=post_data.decode('utf-8')
        print(data)

        if data == 'q':
            raise KeyboardInterrupt("Intended exception to exit webserver")
        
        coords.append(json.loads(data))

        with open(coordinate_filepath, 'w') as json_file:
            json.dump(coords, json_file)

        time.sleep(2)
        last_clicked_coords = coords[-1]
        latitude = last_clicked_coords['latitude']
        longitude = last_clicked_coords['longitude']
        print(last_clicked_coords)    
        js_code = f"alert('process'); map.flyTo([{last_clicked_coords[0]},{last_clicked_coords[1]}],18); "
                
        driver.execute_script(js_code)
        time.sleep(2)
        folder_path=r"C:\Users\jkhus\Desktop\ps\images"

        if not os.path.exists(folder_path):
                os.makedirs(folder_path)
                    
        screenshot_path = os.path.join(folder_path, "1.png")
        driver.save_screenshot(screenshot_path)
        img = cv2.imread(screenshot_path)
        resized = cv2.resize(img, None, fx=0.66,fy=0.66, interpolation=cv2.INTER_AREA)
        # Get height and width of the image
        h, w = img.shape[:2]

        # Display the image
        cv2.imshow( 'image', resized)
        cv2.waitKey(0) 

        # Draw gray box around image to detect edge buildings
        cv2.rectangle(resized, (0, 0), (w - 1, h - 1), (50, 50, 50), 1)

        # Convert image to HSV
        hsv = cv2.cvtColor(resized, cv2.COLOR_BGR2HSV)
        cv2.imshow('hsv',hsv)
        cv2.waitKey(0)
        # Define color ranges(330, 62, 18)
        low_green = (45, 17, 31)
        high_green = (120, 255, 255)

        #define yellow
        low_yellow = (0, 28, 0)
        high_yellow = (27, 255, 255)

        # Create masks
        yellow_mask = cv2.inRange(hsv, low_yellow, high_yellow)
        gray_mask = cv2.inRange(hsv, low_green, high_green)

        # Combine masks
        combined_mask = cv2.bitwise_or(yellow_mask, gray_mask)
        kernel = np.ones((3, 3), dtype=np.uint8)
        combined_mask = cv2.morphologyEx(combined_mask, cv2.MORPH_ERODE, kernel)

        # Find contours
        contours, hier = cv2.findContours(combined_mask, cv2.RETR_TREE, cv2.CHAIN_APPROX_SIMPLE)
        sorted_contours= sorted(contours, key=cv2.contourArea, reverse=True)

        res= sorted_contours[0:10]


        # Draw the outline of all contours
        for cnt in res:
            cv2.drawContours(resized, [cnt], 0, (0, 255, 0), 2)

        # Display result
        cv2.imshow("Result", resized)
        cv2.waitKey(0)
        cv2.destroyAllWindows() 
            
        edges_pil = Image.fromarray(resized)

        # Convert PIL image to bytes
        buffer = BytesIO()
        edges_pil.save(buffer, format="PNG")
        buffer.seek(0)

            # Send the response with the edge-detected image
        self.send_response(200)
        self.send_header('Content-type', 'image/png')
        self.end_headers()
        self.wfile.write(buffer.getvalue())


        

def listen_to_folium_map(port=3001):
    server_address = ('', port)
    httpd = HTTPServer(server_address, FoliumServer)
    print("Server started")
    
    try:
        httpd.serve_forever()
    except KeyboardInterrupt:
        pass

    httpd.server_close()
    print("Server stopped...")


coords = []


if __name__ == "__main__":
    # create variables
    folium_port = 3001
    map_filepath = r"C:\Users\jkhus\Desktop\ps\folium-map.html"
    center_coord = [22.3221, 73.165]
    coordinate_filepath = "coords.json"

    # create folium map
    create_folium_map(map_filepath, center_coord, folium_port)
    
    server_thread = threading.Thread(target=listen_to_folium_map, args=(folium_port,))
    server_thread.start()

    # open the folium map (selenium)
    driver = open_folium_map(map_filepath)

    # print all collected coords
    json.dump(coords, open(coordinate_filepath, 'w'))
    
    last_clicked = load_coords()
    print('last clicked coords: ', last_clicked)
