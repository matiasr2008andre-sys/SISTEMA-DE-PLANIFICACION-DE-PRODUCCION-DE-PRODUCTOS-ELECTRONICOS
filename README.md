# SISTEMA-DE-PLANIFICACION-DE-PRODUCCION-DE-PRODUCTOS-ELECTRONICOS
/* ===================== UTILIDADES DE TEXTO (propias) ===================== */
/*
 * Universidad de las Americas - Facultad de Negocios digitales
 *  - Programacion I
 * Actividad evaluativa RC1: Optimizacion de produccion en una fabrica de
 *                           componentes electronicos
 *
 * Integrantes:
 *   1. Matías Riera
 *   2. Mateo Pule
 *
 * Restricciones cumplidas:
 *   - Sin estructuras (struct) ni archivos: los datos viven solo en memoria.
 *   - Solo arreglos paralelos y punteros.
 *   - Busqueda propia (lineal, sin strcmp/lfind/bsearch/qsort).
 *
 * Datos por producto (arreglos paralelos, mismo indice = mismo producto):
 *   nombres[i]      -> nombre del producto
 *   demanda[i]      -> unidades demandadas
 *   tiempoUnit[i]   -> minutos de fabricacion por unidad
 *   recursoUnit[i]  -> unidades de recurso consumidas por unidad fabricada
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <ctype.h>

#define MAX_PRODUCTOS 5
#define LONG_NOMBRE   50
#define LONG_BUFFER   100

/* ===================== Prototipos ===================== */

/* Entrada de datos validada */
void leerLinea(char *buffer, int tam);
void leerTexto(const char *mensaje, char *destino, int tam);
int  leerEntero(const char *mensaje, int min, int max);
double leerReal(const char *mensaje, double min, double max);
char leerSiNo(const char *mensaje);

/* Utilidades de cadenas y busqueda propias */
void copiarTexto(char *destino, const char *origen, int tam);
int  nombresIguales(const char *a, const char *b);
int  buscarProducto(char nombres[][LONG_NOMBRE], int cantidad, const char *buscado);

/* Operaciones del sistema */
void agregarProducto(char nombres[][LONG_NOMBRE], int *demanda, double *tiempoUnit,
                     double *recursoUnit, int *cantidad);
void listarProductos(char nombres[][LONG_NOMBRE], int *demanda, double *tiempoUnit,
                     double *recursoUnit, int cantidad);
void editarProducto(char nombres[][LONG_NOMBRE], int *demanda, double *tiempoUnit,
                    double *recursoUnit, int cantidad);
void eliminarProducto(char nombres[][LONG_NOMBRE], int *demanda, double *tiempoUnit,
                      double *recursoUnit, int *cantidad);
void configurarLimites(double *tiempoDispHoras, double *recursoDisp);
void evaluarProduccion(char nombres[][LONG_NOMBRE], int *demanda, double *tiempoUnit,
                       double *recursoUnit, int cantidad,
                       double tiempoDispHoras, double recursoDisp);
void mostrarMenu(void);

/* ===================== Entrada de datos ===================== */

/* Lee una linea completa, descarta el exceso y quita el salto de linea. */
void leerLinea(char *buffer, int tam) {
    char *p;
    int c;

    if (fgets(buffer, tam, stdin) == NULL) {
        printf("\nFin de la entrada. Programa terminado.\n");
        exit(0);
    }
    p = buffer;
    while (*p != '\0' && *p != '\n') {
        p++;
    }
    if (*p == '\n') {
        *p = '\0';
    } else {
        /* La linea era mas larga que el buffer: se descarta el resto. */
        while ((c = getchar()) != '\n' && c != EOF) {
        }
    }
}

/* Lee un texto no vacio (sin espacios al inicio ni al final). */
void leerTexto(const char *mensaje, char *destino, int tam) {
    char buffer[LONG_BUFFER];
    char *inicio, *fin;

    while (1) {
        printf("%s", mensaje);
        leerLinea(buffer, LONG_BUFFER);

        inicio = buffer;
        while (*inicio == ' ' || *inicio == '\t') {
            inicio++;
        }
        fin = inicio + strlen(inicio);
        while (fin > inicio && (*(fin - 1) == ' ' || *(fin - 1) == '\t')) {
            fin--;
        }
        *fin = '\0';

        if (*inicio == '\0') {
            printf("  [!] El texto no puede estar vacio.\n");
        } else {
            copiarTexto(destino, inicio, tam);
            return;
        }
    }
}

/* Lee un entero dentro de [min, max]; repite hasta que sea valido. */
int leerEntero(const char *mensaje, int min, int max) {
    char buffer[LONG_BUFFER];
    char *fin;
    long valor;

    while (1) {
        printf("%s", mensaje);
        leerLinea(buffer, LONG_BUFFER);

        valor = strtol(buffer, &fin, 10);
        if (fin == buffer) {
            printf("  [!] Ingrese un numero entero valido.\n");
            continue;
        }
        while (*fin == ' ' || *fin == '\t') {
            fin++;
        }
        if (*fin != '\0') {
            printf("  [!] Ingrese solo un numero entero.\n");
        } else if (valor < min || valor > max) {
            printf("  [!] El valor debe estar entre %d y %d.\n", min, max);
        } else {
            return (int)valor;
        }
    }
}

/* Lee un real dentro de [min, max]; acepta coma o punto decimal. */
double leerReal(const char *mensaje, double min, double max) {
    char buffer[LONG_BUFFER];
    char *fin, *p;
    double valor;

    while (1) {
        printf("%s", mensaje);
        leerLinea(buffer, LONG_BUFFER);

        for (p = buffer; *p != '\0'; p++) {
            if (*p == ',') {
                *p = '.';
            }
        }
        valor = strtod(buffer, &fin);
        if (fin == buffer) {
            printf("  [!] Ingrese un numero valido.\n");
            continue;
        }
        while (*fin == ' ' || *fin == '\t') {
            fin++;
        }
        if (*fin != '\0') {
            printf("  [!] Ingrese solo un numero.\n");
        } else if (!(valor >= min && valor <= max)) {
            printf("  [!] El valor debe estar entre %.2f y %.2f.\n", min, max);
        } else {
            return valor;
        }
    }
}

/* Pregunta s/n y devuelve 's' o 'n'. */
char leerSiNo(const char *mensaje) {
    char buffer[LONG_BUFFER];

    while (1) {
        printf("%s", mensaje);
        leerLinea(buffer, LONG_BUFFER);
        if (buffer[0] != '\0' && buffer[1] == '\0') {
            char c = (char)tolower((unsigned char)buffer[0]);
            if (c == 's' || c == 'n') {
                return c;
            }
        }
        printf("  Responda con 's' o 'n'.\n");
    }
}

/* ===================== Cadenas y busqueda propias ===================== */

/* Copia como maximo tam-1 caracteres y termina en '\0'. */
void copiarTexto(char *destino, const char *origen, int tam) {
    int i = 0;
    while (*(origen + i) != '\0' && i < tam - 1) {
        *(destino + i) = *(origen + i);
        i++;
    }
    *(destino + i) = '\0';
}

/* Compara dos textos sin distinguir mayusculas/minusculas. 1 = iguales. */
int nombresIguales(const char *a, const char *b) {
    while (*a != '\0' && *b != '\0') {
        if (tolower((unsigned char)*a) != tolower((unsigned char)*b)) {
            return 0;
        }
        a++;
        b++;
    }
    return (*a == '\0' && *b == '\0');
}

/* Busqueda lineal propia. Devuelve el indice o -1 si no existe. */
int buscarProducto(char nombres[][LONG_NOMBRE], int cantidad, const char *buscado) {
    int i;
    for (i = 0; i < cantidad; i++) {
        if (nombresIguales(nombres[i], buscado)) {
            return i;
        }
    }
    return -1;
}

/* ===================== Operaciones ===================== */

void agregarProducto(char nombres[][LONG_NOMBRE], int *demanda, double *tiempoUnit,
                     double *recursoUnit, int *cantidad) {
    char nombre[LONG_NOMBRE];

    if (*cantidad >= MAX_PRODUCTOS) {
        printf("\n[!] Ya se registraron los %d productos permitidos.\n", MAX_PRODUCTOS);
        return;
    }

    printf("\n--- INGRESO DE PRODUCTO %d de %d ---\n", *cantidad + 1, MAX_PRODUCTOS);
    leerTexto("Nombre del producto: ", nombre, LONG_NOMBRE);

    if (buscarProducto(nombres, *cantidad, nombre) != -1) {
        printf("[!] Ya existe un producto con ese nombre.\n");
        return;
    }

    copiarTexto(nombres[*cantidad], nombre, LONG_NOMBRE);
    *(demanda + *cantidad)     = leerEntero("Cantidad demandada (unidades): ", 0, 1000000);
    *(tiempoUnit + *cantidad)  = leerReal("Tiempo de fabricacion por unidad (min): ", 0.01, 100000);
    *(recursoUnit + *cantidad) = leerReal("Recursos por unidad (unidades de recurso): ", 0.01, 100000);

    (*cantidad)++;
    printf("\nProducto registrado correctamente.\n");
}

void listarProductos(char nombres[][LONG_NOMBRE], int *demanda, double *tiempoUnit,
                     double *recursoUnit, int cantidad) {
    int i;

    if (cantidad == 0) {
        printf("\n[!] No hay productos registrados.\n");
        return;
    }

    printf("\n--- PRODUCTOS REGISTRADOS (%d de %d) ---\n", cantidad, MAX_PRODUCTOS);
    printf("%-3s %-22s %10s %14s %14s\n", "No.", "Producto", "Demanda",
           "Tiempo/u (min)", "Recurso/u");
    printf("------------------------------------------------------------------\n");
    for (i = 0; i < cantidad; i++) {
        printf("%-3d %-22.22s %10d %14.2f %14.2f\n", i + 1, nombres[i],
               *(demanda + i), *(tiempoUnit + i), *(recursoUnit + i));
    }
}

void editarProducto(char nombres[][LONG_NOMBRE], int *demanda, double *tiempoUnit,
                    double *recursoUnit, int cantidad) {
    char buscado[LONG_NOMBRE], nuevo[LONG_NOMBRE];
    int indice, opcion;

    if (cantidad == 0) {
        printf("\n[!] No hay productos para editar.\n");
        return;
    }

    printf("\n--- EDICION DE PRODUCTO ---\n");
    leerTexto("Nombre del producto a editar: ", buscado, LONG_NOMBRE);
    indice = buscarProducto(nombres, cantidad, buscado);

    if (indice == -1) {
        printf("[!] Producto no encontrado.\n");
        return;
    }

    do {
        printf("\nEditando: %s\n", nombres[indice]);
        printf("  1. Nombre         (actual: %s)\n", nombres[indice]);
        printf("  2. Demanda        (actual: %d)\n", *(demanda + indice));
        printf("  3. Tiempo/unidad  (actual: %.2f min)\n", *(tiempoUnit + indice));
        printf("  4. Recurso/unidad (actual: %.2f)\n", *(recursoUnit + indice));
        printf("  0. Terminar edicion\n");
        opcion = leerEntero("Opcion: ", 0, 4);

        switch (opcion) {
            case 1:
                leerTexto("Nuevo nombre: ", nuevo, LONG_NOMBRE);
                {
                    int otro = buscarProducto(nombres, cantidad, nuevo);
                    if (otro != -1 && otro != indice) {
                        printf("[!] Ya existe otro producto con ese nombre.\n");
                    } else {
                        copiarTexto(nombres[indice], nuevo, LONG_NOMBRE);
                        printf("Nombre actualizado.\n");
                    }
                }
                break;
            case 2:
                *(demanda + indice) = leerEntero("Nueva demanda (unidades): ", 0, 1000000);
                printf("Demanda actualizada.\n");
                break;
            case 3:
                *(tiempoUnit + indice) = leerReal("Nuevo tiempo por unidad (min): ", 0.01, 100000);
                printf("Tiempo actualizado.\n");
                break;
            case 4:
                *(recursoUnit + indice) = leerReal("Nuevo recurso por unidad: ", 0.01, 100000);
                printf("Recurso actualizado.\n");
                break;
            default:
                break;
        }
    } while (opcion != 0);
}

void eliminarProducto(char nombres[][LONG_NOMBRE], int *demanda, double *tiempoUnit,
                      double *recursoUnit, int *cantidad) {
    char buscado[LONG_NOMBRE];
    int indice, i;

    if (*cantidad == 0) {
        printf("\n[!] No hay productos para eliminar.\n");
        return;
    }

    printf("\n--- ELIMINACION DE PRODUCTO ---\n");
    leerTexto("Nombre del producto a eliminar: ", buscado, LONG_NOMBRE);
    indice = buscarProducto(nombres, *cantidad, buscado);

    if (indice == -1) {
        printf("[!] Producto no encontrado.\n");
        return;
    }

    printf("Se eliminara: %s (demanda %d)\n", nombres[indice], *(demanda + indice));
    if (leerSiNo("Confirmar eliminacion (s/n): ") == 'n') {
        printf("Eliminacion cancelada.\n");
        return;
    }

    /* Se desplazan los elementos posteriores una posicion a la izquierda. */
    for (i = indice; i < *cantidad - 1; i++) {
        copiarTexto(nombres[i], nombres[i + 1], LONG_NOMBRE);
        *(demanda + i)     = *(demanda + i + 1);
        *(tiempoUnit + i)  = *(tiempoUnit + i + 1);
        *(recursoUnit + i) = *(recursoUnit + i + 1);
    }
    (*cantidad)--;
    printf("Producto eliminado.\n");
}

void configurarLimites(double *tiempoDispHoras, double *recursoDisp) {
    printf("\n--- LIMITES DE LA LINEA DE PRODUCCION ---\n");
    *tiempoDispHoras = leerReal("Tiempo de produccion disponible (horas): ", 0.01, 1000000);
    *recursoDisp     = leerReal("Recursos disponibles (unidades de recurso): ", 0.01, 100000000);
    printf("\nLimites registrados.\n");
}

void evaluarProduccion(char nombres[][LONG_NOMBRE], int *demanda, double *tiempoUnit,
                       double *recursoUnit, int cantidad,
                       double tiempoDispHoras, double recursoDisp) {
    int i;
    double tiempoProd, recursoProd;
    double tiempoTotal = 0.0, recursoTotal = 0.0;
    double tiempoDispMin, factorTiempo, factorRecurso, factor;

    if (cantidad == 0) {
        printf("\n[!] No hay productos registrados para evaluar.\n");
        return;
    }

    printf("\n=== EVALUACION DE LA PRODUCCION ===\n");
    printf("%-22s %9s %14s %14s\n", "Producto", "Demanda", "Tiempo (min)", "Recursos");
    printf("------------------------------------------------------------\n");
    for (i = 0; i < cantidad; i++) {
        tiempoProd  = *(demanda + i) * *(tiempoUnit + i);
        recursoProd = *(demanda + i) * *(recursoUnit + i);
        tiempoTotal  += tiempoProd;
        recursoTotal += recursoProd;
        printf("%-22.22s %9d %14.2f %14.2f\n", nombres[i], *(demanda + i),
               tiempoProd, recursoProd);
    }
    printf("------------------------------------------------------------\n");
    printf("%-22s %9s %14.2f %14.2f\n", "TOTAL", "", tiempoTotal, recursoTotal);

    printf("\nTiempo total de fabricacion : %.2f min (%.2f h)\n", tiempoTotal, tiempoTotal / 60.0);
    printf("Recursos totales necesarios : %.2f unidades\n", recursoTotal);

    if (tiempoDispHoras <= 0.0 || recursoDisp <= 0.0) {
        printf("\n[!] Aun no se definieron los limites. Use la opcion 5 para\n"
               "    ingresar tiempo y recursos disponibles y verificar la demanda.\n");
        return;
    }

    tiempoDispMin = tiempoDispHoras * 60.0;
    printf("\nTiempo disponible           : %.2f min (%.2f h)\n", tiempoDispMin, tiempoDispHoras);
    printf("Recursos disponibles        : %.2f unidades\n", recursoDisp);

    printf("\n--- RESULTADO ---\n");
    if (tiempoTotal <= tiempoDispMin) {
        printf("Tiempo   : CUMPLE (sobran %.2f min)\n", tiempoDispMin - tiempoTotal);
    } else {
        printf("Tiempo   : NO CUMPLE (faltan %.2f min)\n", tiempoTotal - tiempoDispMin);
    }
    if (recursoTotal <= recursoDisp) {
        printf("Recursos : CUMPLE (sobran %.2f unidades)\n", recursoDisp - recursoTotal);
    } else {
        printf("Recursos : NO CUMPLE (faltan %.2f unidades)\n", recursoTotal - recursoDisp);
    }

    if (tiempoTotal <= tiempoDispMin && recursoTotal <= recursoDisp) {
        printf("\n>>> La fabrica PUEDE cumplir con la demanda.\n");
    } else {
        printf("\n>>> La fabrica NO puede cumplir con la demanda completa.\n");
        factorTiempo  = tiempoDispMin / tiempoTotal;
        factorRecurso = recursoDisp / recursoTotal;
        factor = (factorTiempo < factorRecurso) ? factorTiempo : factorRecurso;
        if (factor > 1.0) {
            factor = 1.0;
        }
        printf("    Con los limites actuales solo se podria atender cerca del %.1f%% de la demanda.\n",
               factor * 100.0);
    }
}

void mostrarMenu(void) {
    printf("\n==========================================\n");
    printf("\n");
    printf("             ||| ||| ||| |||\n");
    printf("          +-------------------+\n");
    printf("       ===|    FABRICA  DE    |===\n");
    printf("       ===|   COMPONENTES     |===\n");
    printf("       ===|   ELECTRONICOS    |===\n");
    printf("          +-------------------+\n");
    printf("             ||| ||| ||| |||\n");
    printf("\n");
    printf("   SISTEMA DE PLANIFICACION DE PRODUCCION\n");
    printf("================================================\n");
    printf("=  1. Ingresar producto                        =\n");
    printf("=  2. Listar productos                         =\n");
    printf("=  3. Editar producto                          =\n");
    printf("=  4. Eliminar producto                        =\n");
    printf("=  5. Configurar tiempo y recursos disponibles:=\n");
    printf("=  6. Evaluar produccion                       =\n");
    printf("=  0. Salir                                    =\n");
    printf("------------------------------------------------\n");
}

/* ===================== Programa principal ===================== */

int main(void) {
    char nombres[MAX_PRODUCTOS][LONG_NOMBRE];
    int demanda[MAX_PRODUCTOS];
    double tiempoUnit[MAX_PRODUCTOS];
    double recursoUnit[MAX_PRODUCTOS];
    int cantidad = 0;
    double tiempoDispHoras = 0.0, recursoDisp = 0.0;
    int opcion;

    do {
        mostrarMenu();
        opcion = leerEntero("Seleccione una opcion: ", 0, 6);

        switch (opcion) {
            case 1:
                agregarProducto(nombres, demanda, tiempoUnit, recursoUnit, &cantidad);
                break;
            case 2:
                listarProductos(nombres, demanda, tiempoUnit, recursoUnit, cantidad);
                break;
            case 3:
                editarProducto(nombres, demanda, tiempoUnit, recursoUnit, cantidad);
                break;
            case 4:
                eliminarProducto(nombres, demanda, tiempoUnit, recursoUnit, &cantidad);
                break;
            case 5:
                configurarLimites(&tiempoDispHoras, &recursoDisp);
                break;
            case 6:
                evaluarProduccion(nombres, demanda, tiempoUnit, recursoUnit, cantidad,
                                  tiempoDispHoras, recursoDisp);
                break;
            case 0:
                printf("\nPrograma finalizado.\n");
                break;
        }
    } while (opcion != 0);

    return 0;
}

