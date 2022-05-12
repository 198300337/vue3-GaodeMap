<script>
import { onMounted, onUnmounted, ref, h } from '@vue/runtime-core'

// pnpoly算法
function isInPolygon(point, vs) {
  const x = point[0]
  const y = point[1]

  let inside = false
  for (let i = 0, j = vs.length - 1; i < vs.length; j = i++) {
    const xi = vs[i][0]
    const yi = vs[i][1]
    const xj = vs[j][0]
    const yj = vs[j][1]

    const intersect =
      yi > y != yj > y && x < ((xj - xi) * (y - yi)) / (yj - yi) + xi
    if (intersect) inside = !inside
  }

  return inside
}

export default {
  name: 'Home',
  render() {
    return h('div', { ref: 'mapContainer', class: 'map' })
  },
  setup(_, { emit }) {
    const mapContainer = ref(null)
    let mapInstance = null

    onUnmounted(() => {
      if (mapInstance) mapInstance.destroy()
    })

    onMounted(async () => {
      mapInstance = new AMap.Map(mapContainer.value, {
        resizeEnable: true,
        center: [117.000923, 36.675807],
        zoom: 11,
        mapStyle: 'amap://styles/c925bb66eb8aa3aefdee13eadeb3ed19'
      })

      const district = new AMap.DistrictSearch({
        extensions: 'all',
        level: 'province',
        subdistrict: 2
      })

      const city = new AMap.DistrictSearch({
        extensions: 'all',
        level: 'city',
        subdistrict: 0
      })

      function cityPromiseFunc(adcode) {
        return new Promise((resolve, reject) => {
          try {
            city.search(adcode, (status, data) => {
              if (status === 'complete') {
                const { name, adcode, boundaries, center, citycode } =
                  data.districtList[0]

                return resolve({
                  status,
                  data: { name, adcode, boundaries, center, citycode }
                })
              }

              resolve({ status, data: {} })
            })
          } catch (error) {
            reject(error)
          }
        })
      }

      const { error, result } = await new Promise((resolve, reject) => {
        try {
          district.search('浙江省', async function (status, result) {
            if (status !== 'complete') {
              resolve({
                error: status,
                result
              })
              return
            }
            resolve({
              error: null,
              result
            })
          })
        } catch (error) {
          reject(error)
        }
      })

      if (error) return

      // 浙江省所有市级区划
      const allCity = result.districtList[0].districtList
      const queueList = allCity.map((e) => {
        return cityPromiseFunc(e.adcode)
      })

      const resList = await Promise.all(queueList)
      const polygons = resList.map((e, i) => {
        const { data } = e

        const polygon = new AMap.Polygon({
          strokeWeight: 2,
          path: data.boundaries,
          fillColor: 'rgba(40, 92, 157, 0.6)',
          strokeColor: 'rgba(78, 170, 232, 0.6)',
          extData: {
            overOption: {
              fillColor: 'rgba(109, 235, 252, 1)'
            },
            outOption: {
              fillColor: 'rgba(40, 92, 157, 0.6)'
            },
            data,
            i
          }
        })

        return polygon
      })

      mapInstance.add(polygons)

      // 鼠标移动事件
      let timeout = null
      let beforePolygon = null
      const boundsMap = new Map()

      mapInstance.on('mousemove', (e) => {
        if (timeout) {
          clearTimeout(timeout)
          timeout = null
        }

        timeout = setTimeout(() => {
          const target = polygons.find((p) => {
            const { i } = p.getExtData()

            let bounds = []
            const cacheBound = boundsMap.get(i)

            if (!cacheBound) {
              const targetBounds = resList[i].data.boundaries
                .map((c) => {
                  return c.map((d) => {
                    const { x, y } = mapInstance.lngLatToContainer([
                      d.lng,
                      d.lat
                    ])
                    return [x, y]
                  })
                })
                .flat(1)
              boundsMap.set(i, targetBounds)
              bounds = targetBounds
            } else {
              bounds = cacheBound
            }

            const { lng, lat } = e.lnglat
            const { x, y } = mapInstance.lngLatToContainer([lng, lat])
            return isInPolygon([x, y], bounds)
          })

          if (!target) return
          // 高亮效果实现
          if (beforePolygon) {
            const extData = beforePolygon.getExtData()
            beforePolygon.setOptions(extData.outOption)
            beforePolygon = null
            emit('onOut', extData)
          }
          const extData = target.getExtData()
          target.setOptions(extData.overOption)
          beforePolygon = target
          emit('onOver', extData)
        }, 100)
      })

      // 地图自适应
      mapInstance.setFitView(polygons, true, [0, 0, 0, 0])
    })

    return {
      mapContainer
    }
  }
}
</script>

<style lang="scss" scoped>
.map {
  width: 100%;
  height: 100%;
  background: url('../assets/images/40.png') no-repeat center / 100% 100% !important;
}
</style>
