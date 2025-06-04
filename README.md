import itertools
from collections import defaultdict
import time

# ===== Lectura de Tablero =====

def cargar_tablero(fileName):
    """Carga el tablero desde un archivo"""
    with open(fileName, "r") as f:
        lineas = []
        max_cols = 0
        for line in f:
            if line.strip():
                fila = line.strip().split()
                lineas.append(fila)
                max_cols = max(max_cols, len(fila))
        
        # Normalizar todas las filas para que tengan el mismo número de columnas
        for i in range(len(lineas)):
            while len(lineas[i]) < max_cols:
                lineas[i].append("X")  # Rellenar con casillas bloqueadas
    
    return lineas

# ===== Modelado CSP Mejorado =====

class KakuroCSP:
    def __init__(self, tablero):
        self.tablero = tablero
        self.variables = set()
        self.constraints = []
        self.domains = {}
        self.variable_constraints = defaultdict(list)  # Mapeo variable -> restricciones
        
        self.procesar_tablero()
        self.propagar_restricciones()

    def procesar_tablero(self):
        """Procesa el tablero y extrae variables y restricciones"""
        filas, columnas = len(self.tablero), len(self.tablero[0])
        
        # Debug: mostrar dimensiones del tablero
        print(f"Dimensiones del tablero: {filas} x {columnas}")
        
        for i in range(filas):
            for j in range(columnas):
                # Verificar que la celda existe
                if j >= len(self.tablero[i]):
                    continue
                    
                celda = self.tablero[i][j]
                
                # Procesar restricciones verticales (Down)
                if "D" in celda:
                    try:
                        if "AD" in celda:
                            # Formato: AD suma_abajo-suma_derecha
                            partes = celda.split("AD")[1].split("-")
                            suma_down = int(partes[0])
                        else:
                            # Formato: D suma
                            suma_down = int(celda.split("D")[1])
                        
                        # Encontrar variables hacia abajo
                        grupo = []
                        x = i + 1
                        while x < filas and j < len(self.tablero[x]) and self.tablero[x][j] == ".":
                            var = f"{x},{j}"
                            grupo.append(var)
                            self.variables.add(var)
                            x += 1
                        
                        if grupo:
                            print(f"Restricción DOWN en ({i},{j}): suma={suma_down}, variables={grupo}")
                            self.agregar_constraint(grupo, suma_down)
                    except (ValueError, IndexError) as e:
                        print(f"Error procesando restricción DOWN en ({i},{j}): {celda} - {e}")

                # Procesar restricciones horizontales (Across)
                if "A" in celda:
                    try:
                        if "AD" in celda:
                            # Formato: AD suma_abajo-suma_derecha
                            partes = celda.split("AD")[1].split("-")
                            suma_across = int(partes[1])
                        else:
                            # Formato: A suma
                            suma_across = int(celda.split("A")[1])
                        
                        # Encontrar variables hacia la derecha
                        grupo = []
                        y = j + 1
                        while y < columnas and y < len(self.tablero[i]) and self.tablero[i][y] == ".":
                            var = f"{i},{y}"
                            grupo.append(var)
                            self.variables.add(var)
                            y += 1
                        
                        if grupo:
                            print(f"Restricción ACROSS en ({i},{j}): suma={suma_across}, variables={grupo}")
                            self.agregar_constraint(grupo, suma_across)
                    except (ValueError, IndexError) as e:
                        print(f"Error procesando restricción ACROSS en ({i},{j}): {celda} - {e}")

    def agregar_constraint(self, variables, suma_objetivo):
        """Agrega una restricción de suma"""
        constraint_id = len(self.constraints)
        self.constraints.append((variables, suma_objetivo))
        
        # Mapear variables a restricciones
        for var in variables:
            self.variable_constraints[var].append(constraint_id)
            if var not in self.domains:
                self.domains[var] = set(range(1, 10))

    def propagar_restricciones(self):
        """Propaga restricciones iniciales para reducir dominios"""
        cambios = True
        while cambios:
            cambios = False
            for constraint_id, (variables, suma) in enumerate(self.constraints):
                n = len(variables)
                
                # Restricción de suma mínima y máxima posible
                min_suma = sum(range(1, n + 1))
                max_suma = sum(range(10 - n, 10))
                
                if suma < min_suma or suma > max_suma:
                    # Restricción imposible
                    for var in variables:
                        self.domains[var] = set()
                    return
                
                # Reducir dominios basándose en límites de suma
                for var in variables:
                    nuevo_dominio = set()
                    for valor in self.domains[var]:
                        # Verificar si es posible completar la suma con este valor
                        otros_min = sum(range(1, n))  # Suma mínima de otros valores
                        otros_max = sum(range(10 - n + 1, 10))  # Suma máxima de otros valores
                        
                        suma_restante = suma - valor
                        if otros_min <= suma_restante <= otros_max:
                            nuevo_dominio.add(valor)
                    
                    if len(nuevo_dominio) < len(self.domains[var]):
                        self.domains[var] = nuevo_dominio
                        cambios = True

    def es_consistente(self, asignacion, var, valor):
        """Verifica si asignar 'valor' a 'var' es consistente"""
        # Verificar todas las restricciones que involucran esta variable
        for constraint_id in self.variable_constraints[var]:
            grupo, suma_objetivo = self.constraints[constraint_id]
            
            # Obtener valores asignados en este grupo
            valores_asignados = []
            variables_sin_asignar = []
            
            for v in grupo:
                if v == var:
                    valores_asignados.append(valor)
                elif v in asignacion:
                    valores_asignados.append(asignacion[v])
                else:
                    variables_sin_asignar.append(v)
            
            # Verificar unicidad
            if len(set(valores_asignados)) != len(valores_asignados):
                return False
            
            # Verificar suma
            suma_actual = sum(valores_asignados)
            
            if len(variables_sin_asignar) == 0:
                # Grupo completo, verificar suma exacta
                if suma_actual != suma_objetivo:
                    return False
            else:
                # Grupo incompleto, verificar que sea posible completar
                if suma_actual >= suma_objetivo:
                    return False
                
                suma_restante = suma_objetivo - suma_actual
                n_restantes = len(variables_sin_asignar)
                
                # Verificar límites de suma restante
                min_posible = sum(range(1, n_restantes + 1))
                max_posible = sum(range(10 - n_restantes, 10))
                
                if suma_restante < min_posible or suma_restante > max_posible:
                    return False
                
                # Verificar que podemos usar valores únicos restantes
                valores_usados = set(valores_asignados)
                valores_disponibles = set(range(1, 10)) - valores_usados
                
                if len(valores_disponibles) < n_restantes:
                    return False
        
        return True

    def seleccionar_variable(self, asignacion):
        """Selecciona la siguiente variable usando MRV (Minimum Remaining Values)"""
        variables_sin_asignar = [v for v in self.variables if v not in asignacion]
        
        if not variables_sin_asignar:
            return None
        
        # Heurística MRV: seleccionar variable con menor dominio
        return min(variables_sin_asignar, key=lambda v: len(self.domains[v]))

    def ordenar_valores(self, var, asignacion):
        """Ordena valores usando LCV (Least Constraining Value)"""
        # Para simplificar, usar orden ascendente
        # En una implementación más sofisticada, se podría contar cuántos valores
        # se eliminan de otras variables
        return sorted(self.domains[var])

    def inferencia(self, var, valor, asignacion):
        """Realiza inferencia después de asignar un valor (Arc Consistency)"""
        inferencias = {}
        
        # Para cada restricción que involucra esta variable
        for constraint_id in self.variable_constraints[var]:
            grupo, suma_objetivo = self.constraints[constraint_id]
            
            # Calcular valores ya asignados
            valores_asignados = []
            variables_sin_asignar = []
            
            for v in grupo:
                if v == var:
                    valores_asignados.append(valor)
                elif v in asignacion:
                    valores_asignados.append(asignacion[v])
                else:
                    variables_sin_asignar.append(v)
            
            if len(variables_sin_asignar) == 1:
                # Solo queda una variable, calcular su valor exacto
                suma_restante = suma_objetivo - sum(valores_asignados)
                var_restante = variables_sin_asignar[0]
                
                if suma_restante in self.domains[var_restante]:
                    # Verificar que no cause conflictos de unicidad
                    if suma_restante not in valores_asignados:
                        inferencias[var_restante] = suma_restante
                    else:
                        return None  # Conflicto de unicidad
                else:
                    return None  # Valor imposible
        
        return inferencias

    def backtrack(self, asignacion):
        """Algoritmo de backtracking con inferencia"""
        if len(asignacion) == len(self.variables):
            return asignacion

        var = self.seleccionar_variable(asignacion)
        if var is None:
            return asignacion

        for valor in self.ordenar_valores(var, asignacion):
            if self.es_consistente(asignacion, var, valor):
                # Hacer asignación
                asignacion[var] = valor
                
                # Realizar inferencia
                inferencias = self.inferencia(var, valor, asignacion)
                
                if inferencias is not None:
                    # Aplicar inferencias
                    for inf_var, inf_val in inferencias.items():
                        asignacion[inf_var] = inf_val
                    
                    # Continuar búsqueda
                    resultado = self.backtrack(asignacion)
                    if resultado is not None:
                        return resultado
                    
                    # Deshacer inferencias
                    for inf_var in inferencias:
                        if inf_var in asignacion:
                            del asignacion[inf_var]
                
                # Deshacer asignación
                if var in asignacion:
                    del asignacion[var]

        return None

    def resolver(self):
        """Resuelve el CSP"""
        print(f"Resolviendo Kakuro con {len(self.variables)} variables y {len(self.constraints)} restricciones")
        
        inicio = time.time()
        solucion = self.backtrack({})
        fin = time.time()
        
        return solucion

# ===== Mostrar solución =====

def imprimir_solucion(tablero, solucion):
    """Imprime la solución del tablero"""
    print("\nSolución:")
    for i, fila in enumerate(tablero):
        fila_str = ""
        for j, celda in enumerate(fila):
            if celda == ".":
                valor = solucion.get(f"{i},{j}", "?")
                fila_str += f"{valor:>3} "
            else:
                # Truncar celdas muy largas para mejor visualización
                celda_display = celda if len(celda) <= 8 else celda[:8]
                fila_str += f"{celda_display:>8} "
        print(fila_str)

def verificar_solucion(tablero, solucion):
    """Verifica que la solución sea correcta"""
    kakuro = KakuroCSP(tablero)
    
    for grupo, suma_objetivo in kakuro.constraints:
        valores = [solucion[var] for var in grupo if var in solucion]
        
        # Verificar que todos los valores estén asignados
        if len(valores) != len(grupo):
            print(f"Error: grupo {grupo} no está completamente asignado")
            return False
        
        # Verificar unicidad
        if len(set(valores)) != len(valores):
            print(f"Error: valores duplicados en grupo {grupo}: {valores}")
            return False
        
        # Verificar suma
        if sum(valores) != suma_objetivo:
            print(f"Error: suma incorrecta en grupo {grupo}. Esperado: {suma_objetivo}, Obtenido: {sum(valores)}")
            return False
    
    print("✓ Solución verificada correctamente")
    return True

# ===== Cargar y Resolver =====

if __name__ == "__main__":
    # Ejemplo de uso
    fileName = "tablero_dificil.txt"
    
    try:
        print(f"\n== Resolviendo tablero: {fileName} ==")
        tablero = cargar_tablero(fileName)
        
        # Mostrar tablero original
        print("\nTablero original:")
        for fila in tablero:
            print(" ".join(f"{celda:>8}" for celda in fila))
        
        # Resolver
        kakuro = KakuroCSP(tablero)
        solucion = kakuro.resolver()
        
        if solucion:
            imprimir_solucion(tablero, solucion)
            verificar_solucion(tablero, solucion)
        else:
            print(".")
            
    except FileNotFoundError:
        print(f"Error: No se pudo encontrar el archivo {fileName}")
        print("Asegúrate de que el archivo existe en el directorio actual.")
