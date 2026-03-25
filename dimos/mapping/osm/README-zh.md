# OpenStreetMap（OSM，开放街图）

这里提供了用于获取和处理 OpenStreetMap 瓦片的功能，包括坐标转换以及基于位置的 VLM（Vision Language Model，视觉语言模型）查询。

## 获取 MapImage

```python
map_image = get_osm_map(LatLon(lat=..., lon=...), zoom_level=18, n_tiles=4)`
```

OSM 瓦片的尺寸为 256x256 像素，因此使用 4 个瓦片时可得到一张 1024x1024 的地图。

你可以将地图上的像素坐标转换为 GPS 位置，也可以反向转换。

```python
>>> map_image.pixel_to_latlon((300, 500))
LatLon(lat=43.58571248, lon=12.23423511)
>>> map_image.latlon_to_pixel(LatLon(lat=43.58571248, lon=12.23423511))
(300, 500)
```

## CurrentLocationMap

这个类会为你的当前位置维护一张合适的上下文地图，以便执行 VLM 查询。

你需要使用当前位置持续更新它；当你偏离地图中心过远时，它会自动重新获取一张新地图。

```python
curr_map = CurrentLocationMap(QwenVlModel())

# Set your latest position.
curr_map.update_position(LatLon(lat=..., lon=...))

# If you want to get back a GPS position of a feature (Qwen gets your current position).
curr_map.query_for_one_position('Where is the closest farmacy?')
# Returns:
#     LatLon(lat=..., lon=...)

# If you also want to get back a description of the result.
curr_map.query_for_one_position_and_context('Where is the closest pharmacy?')
# Returns:
#     (LatLon(lat=..., lon=...), "Lloyd's Pharmacy on Main Street")
```
