# Домашнее задание к занятию 14 «Средство визуализации Grafana»

## Часть 1. Развертывание инфраструктуры

В рамках задания был самостоятельно развернут стек (Grafana, Prometheus, Node Exporter) с помощью `docker-compose`. Для тестирования нотификаций развернут локальный SMTP-сервер Mailpit.

**Файл `docker-compose.yml`:**
```yaml
version: '3.8'

services:
  node-exporter:
    image: prom/node-exporter:latest
    ports:
      - "9100:9100"
    restart: unless-stopped
    # Собирает метрики хоста (CPU, RAM, диски и т.д.)

  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"
    depends_on:
      - node-exporter
    restart: unless-stopped
    # Prometheus собирает метрики и хранит их

  grafana:
    image: grafana/grafana:latest
    container_name: grafana-hw-grafana-1
    ports:
      - "3000:3000"
    environment:
      # Пароль админа в Grafana 
      - GF_SECURITY_ADMIN_PASSWORD=admin

      # SMTP включен
      - GF_SMTP_ENABLED=true
      # Хост и порт Mailpit (внутри сети Docker)
      - GF_SMTP_HOST=mailpit:1025
      # Валидный from_address 
      - GF_SMTP_FROM_ADDRESS=Misha.kochnev@gmail.com
      # Имя отправителя (будет видно в письме)
      - GF_SMTP_FROM_NAME=Grafana Alerts
      # Пропускаем проверку сертификатов (для локальной среды нормально)
      - GF_SMTP_SKIP_VERIFY=true

      # Опционально: чтобы интерфейс был удобнее при локальной работе
      - GF_SERVER_ROOT_URL=http://localhost:3000
      - GF_USERS_ALLOW_SIGN_UP=false

    depends_on:
      - prometheus
    restart: unless-stopped
    # Grafana: дашборды, алерты, отправка писем

  mailpit:
    image: axllent/mailpit:latest
    ports:
      - "1025:1025"  # SMTP порт (на него шлёт Grafana)
      - "8025:8025"  # Веб-интерфейс (тут видны все письма)
    restart: unless-stopped
    # Mailpit: перехватывает письма и показывает их в браузере (без пароля)

```


Задание 1. Подключение Prometheus

В Grafana был добавлен Prometheus в качестве источника данных по адресу http://prometheus:9090.

Скриншот списка Data sources:

![](https://github.com/user-attachments/assets/81146ab4-4fbe-4605-9025-bfa1fb35a491)
![](https://github.com/user-attachments/assets/62ec142f-f8c3-40e6-ab5f-67100db7d947)
![](https://github.com/user-attachments/assets/cd6418d5-47f5-4cad-836f-ff429fcce22d)

Задание 2. Dashboard и PromQL

Был создан Dashboard с четырьмя панелями.

Использованные PromQL-запросы:

Утилизация CPU (в процентах):
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[1m])) * 100)
Load Average (1/5/15):
node_load1 or node_load5 or node_load15
Свободная оперативная память:
node_memory_MemAvailable_bytes
Свободное место на файловой системе:
node_filesystem_avail_bytes
Скриншот созданного Dashboard:

![](https://github.com/user-attachments/assets/3c66eeb0-c850-4c5b-8055-eea1e7affa6c)

Задание 3. Alerting и Нотификации

Для получения уведомлений был настроен локальный SMTP-сервер Mailpit. В Grafana настроен Contact Point (Email) и правило Alert для панели CPU (порог 80%).

Скриншот тестового уведомления в Mailpit (http://localhost:8025):

![](https://github.com/user-attachments/assets/332e3156-c1fa-434e-9f69-2e6f3ee3d05f)
![](https://github.com/user-attachments/assets/a2b5f711-6ebf-45b9-8f6f-25331cc52f2c)

Скриншот итогового Dashboard с активным правилом Alert:

![](https://github.com/user-attachments/assets/b266fa07-be42-4bd0-9637-c71f691a09da)

Задание 4. JSON Model Dashboard

Ниже приведен листинг файла dashboard.json, экспортированного из Grafana.

json код графиков

```
[{
  "apiVersion": "dashboard.grafana.app/v2",
  "kind": "Dashboard",
  "metadata": {
    "name": "adhq4h5"
  },
  "spec": {
    "annotations": [
      {
        "kind": "AnnotationQuery",
        "spec": {
          "builtIn": true,
          "enable": true,
          "hide": true,
          "iconColor": "rgba(0, 211, 255, 1)",
          "name": "Annotations & Alerts",
          "query": {
            "datasource": {
              "name": "-- Grafana --"
            },
            "group": "grafana",
            "kind": "DataQuery",
            "spec": {},
            "version": "v0"
          }
        }
      }
    ],
    "cursorSync": "Off",
    "editable": true,
    "elements": {
      "panel-1": {
        "kind": "Panel",
        "spec": {
          "data": {
            "kind": "QueryGroup",
            "spec": {
              "queries": [
                {
                  "kind": "PanelQuery",
                  "spec": {
                    "hidden": false,
                    "query": {
                      "datasource": {
                        "name": "cg300rwnh2h34c"
                      },
                      "group": "prometheus",
                      "kind": "DataQuery",
                      "spec": {
                        "editorMode": "code",
                        "expr": "100 - (avg by(instance) (rate(node_cpu_seconds_total{mode=\"idle\"}[1m])) * 100)",
                        "legendFormat": "__auto",
                        "range": true
                      },
                      "version": "v0"
                    },
                    "refId": "A"
                  }
                }
              ],
              "queryOptions": {},
              "transformations": []
            }
          },
          "description": "",
          "id": 1,
          "links": [],
          "title": "CPU Utilization (%)",
          "vizConfig": {
            "group": "timeseries",
            "kind": "VizConfig",
            "spec": {
              "fieldConfig": {
                "defaults": {
                  "color": {
                    "mode": "palette-classic"
                  },
                  "custom": {
                    "axisBorderShow": false,
                    "axisCenteredZero": false,
                    "axisColorMode": "text",
                    "axisLabel": "",
                    "axisPlacement": "auto",
                    "barAlignment": 0,
                    "barWidthFactor": 0.6,
                    "drawStyle": "line",
                    "fillOpacity": 0,
                    "gradientMode": "none",
                    "hideFrom": {
                      "legend": false,
                      "tooltip": false,
                      "viz": false
                    },
                    "insertNulls": false,
                    "lineInterpolation": "linear",
                    "lineWidth": 1,
                    "pointSize": 5,
                    "scaleDistribution": {
                      "type": "linear"
                    },
                    "showPoints": "auto",
                    "showValues": false,
                    "spanNulls": false,
                    "stacking": {
                      "group": "A",
                      "mode": "none"
                    },
                    "thresholdsStyle": {
                      "mode": "off"
                    }
                  },
                  "thresholds": {
                    "mode": "absolute",
                    "steps": [
                      {
                        "color": "green",
                        "value": 0
                      },
                      {
                        "color": "red",
                        "value": 80
                      }
                    ]
                  }
                },
                "overrides": []
              },
              "options": {
                "legend": {
                  "calcs": [],
                  "displayMode": "list",
                  "placement": "bottom",
                  "showLegend": true
                },
                "tooltip": {
                  "hideZeros": false,
                  "mode": "single",
                  "sort": "none"
                }
              }
            },
            "version": "13.3.0"
          }
        }
      },
      "panel-2": {
        "kind": "Panel",
        "spec": {
          "data": {
            "kind": "QueryGroup",
            "spec": {
              "queries": [
                {
                  "kind": "PanelQuery",
                  "spec": {
                    "hidden": false,
                    "query": {
                      "datasource": {
                        "name": "cg300rwnh2h34c"
                      },
                      "group": "prometheus",
                      "kind": "DataQuery",
                      "spec": {
                        "editorMode": "code",
                        "expr": "node_load1 or node_load5 or node_load15",
                        "legendFormat": "__auto",
                        "range": true
                      },
                      "version": "v0"
                    },
                    "refId": "A"
                  }
                }
              ],
              "queryOptions": {},
              "transformations": []
            }
          },
          "description": "",
          "id": 2,
          "links": [],
          "title": "Load Average (1/5/15)",
          "vizConfig": {
            "group": "timeseries",
            "kind": "VizConfig",
            "spec": {
              "fieldConfig": {
                "defaults": {
                  "color": {
                    "mode": "palette-classic"
                  },
                  "custom": {
                    "axisBorderShow": false,
                    "axisCenteredZero": false,
                    "axisColorMode": "text",
                    "axisLabel": "",
                    "axisPlacement": "auto",
                    "barAlignment": 0,
                    "barWidthFactor": 0.6,
                    "drawStyle": "line",
                    "fillOpacity": 0,
                    "gradientMode": "none",
                    "hideFrom": {
                      "legend": false,
                      "tooltip": false,
                      "viz": false
                    },
                    "insertNulls": false,
                    "lineInterpolation": "linear",
                    "lineWidth": 1,
                    "pointSize": 5,
                    "scaleDistribution": {
                      "type": "linear"
                    },
                    "showPoints": "auto",
                    "showValues": false,
                    "spanNulls": false,
                    "stacking": {
                      "group": "A",
                      "mode": "none"
                    },
                    "thresholdsStyle": {
                      "mode": "off"
                    }
                  },
                  "thresholds": {
                    "mode": "absolute",
                    "steps": [
                      {
                        "color": "green",
                        "value": 0
                      },
                      {
                        "color": "red",
                        "value": 80
                      }
                    ]
                  }
                },
                "overrides": []
              },
              "options": {
                "legend": {
                  "calcs": [],
                  "displayMode": "list",
                  "placement": "bottom",
                  "showLegend": true
                },
                "tooltip": {
                  "hideZeros": false,
                  "mode": "single",
                  "sort": "none"
                }
              }
            },
            "version": "13.3.0"
          }
        }
      },
      "panel-3": {
        "kind": "Panel",
        "spec": {
          "data": {
            "kind": "QueryGroup",
            "spec": {
              "queries": [
                {
                  "kind": "PanelQuery",
                  "spec": {
                    "hidden": false,
                    "query": {
                      "datasource": {
                        "name": "cg300rwnh2h34c"
                      },
                      "group": "prometheus",
                      "kind": "DataQuery",
                      "spec": {
                        "editorMode": "code",
                        "expr": "node_memory_MemAvailable_bytes",
                        "legendFormat": "__auto",
                        "range": true
                      },
                      "version": "v0"
                    },
                    "refId": "A"
                  }
                }
              ],
              "queryOptions": {},
              "transformations": []
            }
          },
          "description": "",
          "id": 3,
          "links": [],
          "title": "Free RAM",
          "vizConfig": {
            "group": "timeseries",
            "kind": "VizConfig",
            "spec": {
              "fieldConfig": {
                "defaults": {
                  "color": {
                    "mode": "palette-classic"
                  },
                  "custom": {
                    "axisBorderShow": false,
                    "axisCenteredZero": false,
                    "axisColorMode": "text",
                    "axisLabel": "",
                    "axisPlacement": "auto",
                    "barAlignment": 0,
                    "barWidthFactor": 0.6,
                    "drawStyle": "line",
                    "fillOpacity": 0,
                    "gradientMode": "none",
                    "hideFrom": {
                      "legend": false,
                      "tooltip": false,
                      "viz": false
                    },
                    "insertNulls": false,
                    "lineInterpolation": "linear",
                    "lineWidth": 1,
                    "pointSize": 5,
                    "scaleDistribution": {
                      "type": "linear"
                    },
                    "showPoints": "auto",
                    "showValues": false,
                    "spanNulls": false,
                    "stacking": {
                      "group": "A",
                      "mode": "none"
                    },
                    "thresholdsStyle": {
                      "mode": "off"
                    }
                  },
                  "thresholds": {
                    "mode": "absolute",
                    "steps": [
                      {
                        "color": "green",
                        "value": 0
                      },
                      {
                        "color": "red",
                        "value": 80
                      }
                    ]
                  },
                  "unit": "bytes"
                },
                "overrides": []
              },
              "options": {
                "legend": {
                  "calcs": [],
                  "displayMode": "list",
                  "placement": "bottom",
                  "showLegend": true
                },
                "tooltip": {
                  "hideZeros": false,
                  "mode": "single",
                  "sort": "none"
                }
              }
            },
            "version": "13.3.0"
          }
        }
      },
      "panel-4": {
        "kind": "Panel",
        "spec": {
          "data": {
            "kind": "QueryGroup",
            "spec": {
              "queries": [
                {
                  "kind": "PanelQuery",
                  "spec": {
                    "hidden": false,
                    "query": {
                      "datasource": {
                        "name": "cg300rwnh2h34c"
                      },
                      "group": "prometheus",
                      "kind": "DataQuery",
                      "spec": {
                        "editorMode": "code",
                        "expr": "node_filesystem_avail_bytes",
                        "legendFormat": "__auto",
                        "range": true
                      },
                      "version": "v0"
                    },
                    "refId": "A"
                  }
                }
              ],
              "queryOptions": {},
              "transformations": []
            }
          },
          "description": "",
          "id": 4,
          "links": [],
          "title": "Free Disk Space",
          "vizConfig": {
            "group": "timeseries",
            "kind": "VizConfig",
            "spec": {
              "fieldConfig": {
                "defaults": {
                  "color": {
                    "mode": "palette-classic"
                  },
                  "custom": {
                    "axisBorderShow": false,
                    "axisCenteredZero": false,
                    "axisColorMode": "text",
                    "axisLabel": "",
                    "axisPlacement": "auto",
                    "barAlignment": 0,
                    "barWidthFactor": 0.6,
                    "drawStyle": "line",
                    "fillOpacity": 0,
                    "gradientMode": "none",
                    "hideFrom": {
                      "legend": false,
                      "tooltip": false,
                      "viz": false
                    },
                    "insertNulls": false,
                    "lineInterpolation": "linear",
                    "lineWidth": 1,
                    "pointSize": 5,
                    "scaleDistribution": {
                      "type": "linear"
                    },
                    "showPoints": "auto",
                    "showValues": false,
                    "spanNulls": false,
                    "stacking": {
                      "group": "A",
                      "mode": "none"
                    },
                    "thresholdsStyle": {
                      "mode": "off"
                    }
                  },
                  "thresholds": {
                    "mode": "absolute",
                    "steps": [
                      {
                        "color": "green",
                        "value": 0
                      },
                      {
                        "color": "red",
                        "value": 80
                      }
                    ]
                  }
                },
                "overrides": []
              },
              "options": {
                "legend": {
                  "calcs": [],
                  "displayMode": "list",
                  "placement": "bottom",
                  "showLegend": true
                },
                "tooltip": {
                  "hideZeros": false,
                  "mode": "single",
                  "sort": "none"
                }
              }
            },
            "version": "13.3.0"
          }
        }
      }
    },
    "layout": {
      "kind": "AutoGridLayout",
      "spec": {
        "columnWidthMode": "standard",
        "items": [
          {
            "kind": "AutoGridLayoutItem",
            "spec": {
              "element": {
                "kind": "ElementReference",
                "name": "panel-1"
              }
            }
          },
          {
            "kind": "AutoGridLayoutItem",
            "spec": {
              "element": {
                "kind": "ElementReference",
                "name": "panel-2"
              }
            }
          },
          {
            "kind": "AutoGridLayoutItem",
            "spec": {
              "element": {
                "kind": "ElementReference",
                "name": "panel-3"
              }
            }
          },
          {
            "kind": "AutoGridLayoutItem",
            "spec": {
              "element": {
                "kind": "ElementReference",
                "name": "panel-4"
              }
            }
          }
        ],
        "maxColumnCount": 3,
        "rowHeightMode": "standard"
      }
    },
    "links": [],
    "liveNow": false,
    "preferences": {
      "layout": {
        "kind": "AutoGridLayout",
        "spec": {
          "columnWidthMode": "standard",
          "items": [],
          "maxColumnCount": 3,
          "rowHeightMode": "standard"
        }
      }
    },
    "preload": false,
    "tags": [],
    "timeSettings": {
      "autoRefresh": "",
      "autoRefreshIntervals": [
        "5s",
        "10s",
        "30s",
        "1m",
        "5m",
        "15m",
        "30m",
        "1h",
        "2h",
        "1d"
      ],
      "fiscalYearStartMonth": 0,
      "from": "now-6h",
      "hideTimepicker": false,
      "timezone": "browser",
      "to": "now"
    },
    "title": "New dashboard",
    "variables": []
  }
}]

```


