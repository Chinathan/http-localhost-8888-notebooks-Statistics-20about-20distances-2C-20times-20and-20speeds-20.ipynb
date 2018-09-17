# http-localhost-8888-notebooks-Statistics-20about-20distances-2C-20times-20and-20speeds-20.ipynb

from datetime import *
import osr
import csv
from csv import *
import numpy as np
from numpy import *
from osgeo import ogr
import matplotlib.pyplot as plt
import pandas as pd
import csv
import os
import scipy
from scipy import spatial

def statistics_about_travelled_distances_barplot(path_of_the_raw_file, name_of_the_new_file):

    cleaning_paths=[]
    wgs_84_paths=[]
    epsg_4479_paths=[]
    lengths_of_paths=[]
    times=[]
    lengths_lines_straight=[]
    speed=[]
    speed_int=[]
    speed_straight=[]
    speed_tracks_ordered=[]
    speed_tracks_ordered_int=[]
    lengths_of_ordered_paths=[]
    tracks_ordered=[]

    with open(name_of_the_new_file+'_gcj02.csv','w') as myfile:
        wr=csv.writer(myfile)

        with open(path_of_the_raw_file) as fp:
            for index, line in enumerate(fp):
                if index ==0 :
                    print('index=0')
            
                elif '\\' in line:
                    print('found a \ at line',index)
        
                elif '\.'in line :
                    print('found a . at line',index)
            
                else :
            
                    path_index=[]
                    head_index=line.split(',') #Create string each time there is a ","
                    
                    #Stocking times of each path 
                    starting_time_index=datetime(int(head_index[3][0:4]),int(head_index[3][5:7]), int(head_index[3][8:10]), int(head_index[3][11:13]),int(head_index[3][14:16]))
                    ending_time_index=datetime(int(head_index[6][0:4]),int(head_index[6][5:7]), int(head_index[6][8:10]), int(head_index[6][11:13]),int(head_index[6][14:16]))
                    delta_time=ending_time_index-starting_time_index
                    times.append(delta_time)
                    
                    try:
                        first_intermediary_longitude=head_index[9].replace("\"","") # line contains the latitude of the ending point then 'lat_end_point','""121.485','31.275#121.485','31.276#121.486
            #need to remove specific characters like " 
                    except IndexError:
                        pass
                    
                    first_intermediary_latitude=head_index[10].split('#')[0] 
                    try :
                        first_intermediary_coordinate=(float(first_intermediary_longitude),float(first_intermediary_latitude))
                    except ValueError:
                        print(first_intermediary_longitude,first_intermediary_latitude)
                        pass
                    del head_index[9:] #remove the strings from ninth position to the end to obtain starting and ending points only
                    try:
                        start_index=(float(head_index[4]), float(head_index[5]))#tranform the string containing the starting point of the index path in a couple of integer
                        end_index=(float(head_index[-2]),float(head_index[-1])) #transform the string containing the ending point of the index path in a couple of integer
                    except ValueError:
                        print(head_index[4],head_index[5])
                        pass
                    try:
                        track_index=line.split('#') #obtaining the intermediaries coordinates of the tracks which are seperated by'#'
                    except IndexError:
                        print(lin)
                        pass
                    del track_index[0]
                    track_index[-1]=track_index[-1].replace('"','') 
                    for i in track_index:
                        path_index.append(eval(i))
            
                    path_index.insert(0, start_index)
                    path_index.insert(1,first_intermediary_coordinate)
                    path_index.append(end_index)
                    wr.writerow(path_index)
                    cleaning_paths.append(path_index)


    with open(name_of_the_new_file+'_wgs84.csv','w') as myfile:
        wr=csv.writer(myfile)
    
        for k in range(0,len(cleaning_paths)): 
            path_k=cleaning_paths[k]
            wgs_84_path_k=[] #Be careful, do not mistake with wgs84_paths whose type is a list of lists
    
            for coordinate in path_k: # from couples in gcj geodetic system to couples in wgs-84 
                shifted_coordinate=gcj02_to_wgs84(coordinate[0],coordinate[1])[0],gcj02_to_wgs84(coordinate[0],coordinate[1])[1]
                wgs_84_path_k.append(shifted_coordinate)
            wr.writerow(wgs_84_path_k)
            wgs_84_paths.append(wgs_84_path_k)

    with open(name_of_the_new_file+'_epsg4479.csv','w') as myfile:
        wr=csv.writer(myfile)

        for i in wgs_84_paths :
            line_i=ogr.Geometry(ogr.wkbLineString) #create a linestring in which we gonna add euclidian coodinates to then easily calculate distance of each path
            line_i_start_stop=ogr.Geometry(ogr.wkbLineString) #create a linestring to then determine the speed between start and stop in straight line
            
            epsg_4479_path_i=batchConvertCoordinates(i,4326,4479) #projection of spheric coordinates in euclidian coordinates
            wr.writerow(epsg_4479_path_i)
            epsg_4479_paths.append(epsg_4479_path_i)
            
            line_i_start_stop.AddPoint(epsg_4479_path_i[0][0], epsg_4479_path_i[0][1])
            line_i_start_stop.AddPoint(epsg_4479_path_i[-1][0], epsg_4479_path_i[-1][1])
            lengths_lines_straight.append(line_i_start_stop.Length())
    
            for j in epsg_4479_path_i: 
                line_i.AddPoint(j[0],j[1]) #add euclidian coordinates 
            
   
            length_line_i=line_i.Length() #calculate the distance of the linestring
            #if length_line_i<3977: #Remove tracks whose length is less than the third quartile 
            lengths_of_paths.append(length_line_i)
        
    with open(name_of_the_new_file+'_epsg4479_reordered.csv','w') as myfile:
        wr=csv.writer(myfile)
        
        for track in epsg_4479_paths:
            track_ordered=[]
            start=track[0]
            stop=track[-1]
            track_ordered.append(start)
            track=track[1:len(track)]
            tree=spatial.KDTree(track)
            i=start
            while len(track)>1:
                idx=tree.query(i)[1] #stock the index corresponding to the closest point to 'i'
                track_ordered.append(track[idx])
                i=track[idx]
                del track[idx]
                tree=spatial.KDTree(track)
            track_ordered.append(stop) #A list of ordered coordinates (for only one track)
            wr.writerow(track_ordered)
            tracks_ordered.append(track_ordered) # A list of list of ordered coordinates
        
        for track in tracks_ordered: 
            line_track=ogr.Geometry(ogr.wkbLineString)
            for j in track:
                line_track.AddPoint(j[0],j[1])
            lengths_of_ordered_paths.append(line_track.Length())
            
        
            
        print(len(times),len(lengths_of_paths),len(lengths_lines_straight))
        for index, k in enumerate(times):
            speed_value=lengths_of_paths[index]/(k.seconds/3600)
            speed_straight_value=lengths_lines_straight[index]/(k.seconds/3600)
            speed_value_ordered_tracks=lengths_of_ordered_paths[index]/(k.seconds/3600)
            speed.append(speed_value)
            speed_straight.append(speed_straight_value)
            speed_tracks_ordered.append(speed_value_ordered_tracks)
            speed_int.append(int(speed_value/1000))
            speed_tracks_ordered_int.append(int(speed_value_ordered_tracks/1000))
            
    
    #How to display overlayed barchart
    
    dict_1=count_liste(speed_int) #dictionnary of apparitions of speeds before tracks have been ordered
    dict_2=count_liste(speed_tracks_ordered_int) #dictionnary of apparitions of speed after tracks have been ordered 
    
    speeds_before=((dict_1[1]+dict_1[2]+dict_1[3]+dict_1[4]),(dict_1[5]+dict_1[6]+dict_1[7]+dict_1[8]),(dict_1[9]+dict_1[10]+dict_1[11]+dict_1[12]),(dict_1[13]+dict_1[14]+dict_1[15]+dict_1[16]),(dict_1[17]+dict_1[18]+dict_1[19]+dict_1[20]),(dict_1[21]+dict_1[22]+dict_1[23]+dict_1[24]),(dict_1[25]+dict_1[26]+dict_1[27]+dict_1[28]))
    speeds_after=((dict_2[1]+dict_2[2]+dict_2[3]+dict_2[4]),(dict_2[5]+dict_2[6]+dict_2[7]+dict_2[8]),(dict_2[9]+dict_2[10]+dict_2[11]+dict_2[12]),(dict_2[13]+dict_2[14]+dict_2[15]+dict_2[16]),(dict_2[17]+dict_2[18]+dict_2[19]+dict_2[20]),(dict_2[21]+dict_2[22]+dict_2[23]+dict_2[24]),(dict_2[25]+dict_2[26]+dict_2[27]+dict_2[28]))

    fig,ax = plt.subplots()
    fig.set_size_inches(15,15)
    index=np.arange(7)
    bar_width=0.35
    opacity=0.3
    
    rect1=plt.bar(index,speeds_before,bar_width,alpha=opacity,color='grey',label='speeds before having ordered the tracks')
    rect2=plt.bar(index+bar_width,speeds_after,bar_width,alpha=opacity,color='blue',label='speeds after having ordered the tracks')
    plt.xlabel('speeds (km/h)',fontsize=15)
    plt.ylabel('number of apparitions',fontsize=15)
    plt.title('speeds distribution before and after having ordered geodatas provided by MoBike',fontsize=30)
    plt.xticks(index+bar_width/2,('[1,4]','[5,8]','[9,12]','[13,16]','[17,20]','[21,24]','[25,28]'),fontsize=20)
    plt.legend(fontsize=30)
    plt.tight_layout()
    plt.show()
    
    #Statistics
    
    s=pd.Series(lengths_of_paths)
    speed_serie= pd.Series(speed)
    speed_straight_serie=pd.Series(speed_straight)
    speed_ordered_serie=pd.Series(speed_tracks_ordered)
    
    return('lengths of the tracks', s.describe(),'speed serie',speed_serie.describe(),'speed straigt serie', speed_straight_serie.describe(),'speed ordered tracks',speed_ordered_serie.describe())
    
   

    
