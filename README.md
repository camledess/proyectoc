from selenium import webdriver
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from datetime import datetime
import pandas as pd
import time

# Configuración
CHROMEDRIVER_PATH = "chromedriver.exe"
URL_PANEL = "https://admin.celuapuestas.xyz"
FECHA_DESDE = "01/06/2025"
FECHA_HASTA = datetime.today().strftime("%d/%m/%Y")
EXCEL_PATH = "movimientos.xlsx"

# Iniciar navegador
options = Options()
options.add_argument("--start-maximized")
driver = webdriver.Chrome(service=Service(CHROMEDRIVER_PATH), options=options)

# Ir al panel
driver.get(URL_PANEL)
input("🟡 Iniciá sesión y presioná Enter cuando estés en la tabla de jugadores...")

# Esperar la tabla
WebDriverWait(driver, 15).until(
    EC.presence_of_element_located((By.CSS_SELECTOR, "table tbody tr"))
)

# Capturar los primeros 20 usuarios visibles
usuarios = [
    fila.find_elements(By.TAG_NAME, "td")[1].text.strip()
    for fila in driver.find_elements(By.CSS_SELECTOR, "table tbody tr")[:20]
]

# Crear lista de datos
data = []

for usuario in usuarios:
    print(f"\n▶ Jugador detectado: {usuario}")

    # Buscar la fila actualizada del usuario
    WebDriverWait(driver, 10).until(
        EC.presence_of_element_located((By.CSS_SELECTOR, "table tbody tr"))
    )
    filas = driver.find_elements(By.CSS_SELECTOR, "table tbody tr")
    fila = next((f for f in filas if usuario in f.text), None)

    if not fila:
        print(f"⚠️ No se encontró la fila del jugador '{usuario}', se saltea.")
        data.append({"Usuario": usuario, "Último movimiento": "NO SE ENCONTRÓ FILA"})
        continue

    # Abrir menú de acciones
    menu_btns = fila.find_elements(By.CSS_SELECTOR, "td:last-child button")
    if not menu_btns:
        print(f"⚠️ No se encontró botón de menú para el jugador '{usuario}', se saltea.")
        data.append({"Usuario": usuario, "Último movimiento": "NO SE ENCONTRÓ BOTÓN DE MENÚ"})
        continue

    menu_btn = menu_btns[-1]
    driver.execute_script("arguments[0].click();", menu_btn)
    time.sleep(0.5)

    # Esperar que cargue la pantalla de movimientos
    WebDriverWait(driver, 10).until(
        EC.presence_of_element_located((By.XPATH, "//h3[contains(text(), 'Efectivo')]"))
    )
    print("✅ Ya estás en el menú de 'Ultimos Movimientos'")

    # Completar fechas y usuario
    WebDriverWait(driver, 10).until(EC.presence_of_element_located((By.CSS_SELECTOR, "input[formcontrolname='dateFrom']"))).clear()
    driver.find_element(By.CSS_SELECTOR, "input[formcontrolname='dateFrom']").send_keys(FECHA_DESDE)

    WebDriverWait(driver, 10).until(EC.presence_of_element_located((By.CSS_SELECTOR, "input[formcontrolname='dateTo']"))).clear()
    driver.find_element(By.CSS_SELECTOR, "input[formcontrolname='dateTo']").send_keys(FECHA_HASTA)

    input_usuario = WebDriverWait(driver, 10).until(
        EC.presence_of_element_located((By.CSS_SELECTOR, "input[formcontrolname='username']"))
    )
    input_usuario.clear()
    input_usuario.send_keys(usuario)

    # Buscar movimientos
    WebDriverWait(driver, 5).until(
        EC.element_to_be_clickable((By.XPATH, "//button[contains(text(), 'Jugadores')]"))
    ).click()
    WebDriverWait(driver, 5).until(
        EC.element_to_be_clickable((By.CSS_SELECTOR, "button.search-btn"))
    ).click()
    print("🔍 Búsqueda realizada correctamente.")
    time.sleep(2)

    # Revisar si hay resultados
    texto = ""
    try:
        tabla_mov = driver.find_element(By.CSS_SELECTOR, "table tbody")
        filas_mov = tabla_mov.find_elements(By.TAG_NAME, "tr")

        if len(filas_mov) == 1:
            texto_fila = filas_mov[0].text.strip().lower()
            if "sin resultados" in texto_fila or "no hay" in texto_fila:
                texto = "NO ACTIVO"
            else:
                columnas = filas_mov[0].find_elements(By.TAG_NAME, "td")
                if len(columnas) >= 6:
                    fecha = columnas[1].text.strip()
                    tipo = columnas[2].text.strip().upper()
                    monto = columnas[5].text.strip()
                    texto = f"ACTIVO - Último {tipo.lower()} el {fecha} - Cargó ${monto}"
                else:
                    texto = "ACTIVO - Último movimiento disponible"
        elif len(filas_mov) == 0:
            texto = "NO ACTIVO"
        else:
            columnas = filas_mov[0].find_elements(By.TAG_NAME, "td")
            if len(columnas) >= 6:
                fecha = columnas[1].text.strip()
                tipo = columnas[2].text.strip().upper()
                monto = columnas[5].text.strip()
                texto = f"ACTIVO - Último {tipo.lower()} el {fecha} - Cargó ${monto}"
            else:
                texto = "ACTIVO - Último movimiento disponible"
    except Exception as e:
        texto = f"ERROR obteniendo movimientos: {str(e)}"

    # Guardar resultado
    data.append({"Usuario": usuario, "Último movimiento": texto})

    # Volver al menú principal
    driver.get(URL_PANEL)
    WebDriverWait(driver, 10).until(
        EC.presence_of_element_located((By.CSS_SELECTOR, "table tbody tr"))
    )
    print("↩️ Volviste al listado de jugadores.")

# Guardar resultados en Excel
pd.DataFrame(data).to_excel(EXCEL_PATH, index=False)
print("✅ Excel generado correctamente con los primeros jugadores.")

# Cerrar navegador
input("🟢 Proceso completo. Presioná Enter para cerrar...")
driver.quit()
