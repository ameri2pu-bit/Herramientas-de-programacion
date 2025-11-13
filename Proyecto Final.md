import pygame, sys, random  # importa módulos para gráficos, salida y números aleatorios
import matplotlib.pyplot as plt  # importa pyplot de matplotlib para dibujar gráficos

from matplotlib.backends.backend_agg import FigureCanvasAgg  # backend para convertir figuras matplotlib a imágenes


plt.ioff()  # desactiva la interacción automática de figuras
pygame.init()  # inicializa módulos de pygame
W, H = 1024, 768  # ancho y alto de la ventana
screen = pygame.display.set_mode((W, H))  # crea la ventana
pygame.display.set_caption('Blackjack - Minimal')  # establece el título de la ventana

GREEN = (53,101,77); WHITE = (255,255,255); BLACK=(0,0,0)  # colores principales
RED=(255,0,0); GOLD=(255,215,0)  # colores adicionales

# Función para obtener una fuente que soporte símbolos de naipe
def get_font(sz):
    '''fuentes de sistema que incluyen símbolos'''
    for name in ("Segoe UI Symbol","Segoe UI Emoji","Arial Unicode MS","DejaVu Sans"):
        try:
            f = pygame.font.SysFont(name, sz)  # intenta cargar la fuente del sistema
            t = f.render('♠', True, BLACK)  # intenta renderizar un símbolo para validar
            if t.get_width()>0: return f  # si se pudo renderizar, devuelve la fuente
        except Exception:
            pass  # ignora errores y prueba la siguiente fuente
    return pygame.font.Font(None, sz)  # fallback a la fuente por defecto si ninguna funcionó


font = get_font(28); small_font = get_font(20)  # fuente grande y pequeña

# Dibuja un globo de texto con fondo redondeado
def draw_bubble(surf, x, y, text, bg=WHITE, fg=BLACK, alpha=220, r=12):
    px,py = 12,8  # padding horizontal y vertical
    txt = small_font.render(text, True, fg)  # Muestra el texto
    w,h = txt.get_width()+px*2, txt.get_height()+py*2  # calcula tamaño del globo
    s = pygame.Surface((w+4,h+4), pygame.SRCALPHA)  # superficie con alpha para sombra
    shadow = (0,0,0,80)  # color sombra semi-transparente
    try:
        pygame.draw.rect(s, shadow, (2,2,w,h), border_radius=r)  # dibuja sombra con borde redondeado
    except TypeError:
        pygame.draw.rect(s, shadow, (2,2,w,h))  
    bgc = (bg[0],bg[1],bg[2],alpha)  # color de fondo con alpha
    try:
        pygame.draw.rect(s, bgc, (0,0,w,h), border_radius=r)  # dibuja fondo con borde redondeado
        pygame.draw.rect(s, BLACK, (0,0,w,h), 2, border_radius=r)  # dibuja borde externo
    except TypeError:
        pygame.draw.rect(s, bgc, (0,0,w,h))  
        pygame.draw.rect(s, BLACK, (0,0,w,h), 2)  # dibuja borde externo 
    surf.blit(s, (x-2, y-2))  # pega la superficie del globo en la superficie principal
    surf.blit(txt, (x+px, y+py))  # pega el texto en el globo

# Valores de fichas disponibles
CHIP_VALUES = [25, 100, 1000, 5000]  

# Clase para el mazo de cartas
class Deck:
    def __init__(self, num_decks=4):
        suits = ['♠','♥','♦','♣']  # palos
        ranks = ['A','2','3','4','5','6','7','8','9','10','J','Q','K']  # valores
        self.cards = [Card(s,v) for s in suits for v in ranks] * num_decks  # crea cartas multiplicadas por número de barajas
        random.shuffle(self.cards)  # baraja las cartas
    def draw(self):
        if not self.cards: self.__init__()  # si se vacía el mazo, re-inicializa
        return self.cards.pop()  # devuelve la última carta

# Clase que representa una carta individual
class Card:
    def __init__(self, suit, value):
        self.suit = suit; self.value = value; self.shown = True  # palo, valor y si está boca arriba
        self.image = pygame.Surface((80,120)); self.image.fill(WHITE)  # crea imagen de la carta
        pygame.draw.rect(self.image, BLACK, (0,0,80,120), 2) 
        color = RED if suit in ('♥','♦') else BLACK  # color rojo para corazones/diamantes
        self.image.blit(font.render(str(value), True, color), (6,6))  # dibujo del valor en la carta
        self.image.blit(font.render(suit, True, color), (6,36))  # dibujo del símbolo del palo
    def draw(self, surf, pos):
        x,y = pos  
        if not self.shown:
            back = pygame.Surface((80,120)); back.fill(RED); pygame.draw.rect(back, BLACK, (0,0,80,120), 2); surf.blit(back, (x,y))
            # si no está mostrada, dibuja la parte de atrás
        else:
            surf.blit(self.image, (x,y))  # si está mostrada, dibuja la imagen frontal

# Clase para los botones en pantalla
class Button:
    def __init__(self, x,y,w,h,text,color=WHITE,is_chip=False):
        self.rect = pygame.Rect(x,y,w,h); self.text = str(text); self.color = color; self.enabled = True; self.is_chip = is_chip
        # crea rectángulo, texto, color, estado habilitado y si es ficha
    def draw(self, surf):
        col = self.color if self.enabled else (128,128,128)  # color gris si está deshabilitado
        if self.is_chip:
            pygame.draw.circle(surf, col, self.rect.center, self.rect.w//2); pygame.draw.circle(surf, BLACK, self.rect.center, self.rect.w//2, 2)
            # dibuja ficha circular y borde
        else:
            pygame.draw.rect(surf, col, self.rect); pygame.draw.rect(surf, BLACK, self.rect, 2)
            # dibuja rectángulo y borde
        surf.blit(small_font.render(self.text, True, BLACK), small_font.render(self.text, True, BLACK).get_rect(center=self.rect.center))
        # dibuja el texto centrado en el rectángulo/ficha

# Clase principal del juego
class Game:
    def __init__(self):
        self.reset_game()  # inicializa valores de juego
        self.stats = {'manos_jugadas':0, 'dinero_apostado':0, 'dinero_ganado':0, 'dinero_perdido':0, 'historial_dinero':[(0,1000)]}
        # estadísticas iniciales
        btn_w,btn_h,spacing = 100,40,24; cx = W//2  # dimensiones de botones y centro X
        left = cx - (btn_w*3 + spacing*2)//2  # posición izquierda para alinear botones
        self.double_button = Button(left,650,btn_w,btn_h,'Double'); self.hit_button = Button(left+(btn_w+spacing),650,btn_w,btn_h,'Hit')
        # botones Double y Hit
        self.stand_button = Button(left+2*(btn_w+spacing),650,btn_w,btn_h,'Stand')  # botón Stand
        self.deal_button = Button(left+3*(btn_w+spacing)+40,650,btn_w,btn_h,'Deal',color=GOLD)  # botón Deal resaltado
        self.split_button = Button(left-(btn_w+spacing),650,btn_w,btn_h,'Split')  # botón Split
        self.insurance_button = Button(cx+50,140,btn_w,btn_h,'Seguro')  # botón Seguro (insurance)
        self.reset_button = Button(cx-btn_w//2,360,btn_w,btn_h,'Reset', color=GOLD)  # botón Reset
        self.stats_button = Button(W-120,10,100,30,'Stats')  # botón para ver estadísticas
        chip_w,chip_spacing = 50,70; total = len(CHIP_VALUES)*chip_spacing; start = cx - total//2
        # crea botones de fichas centrados
        self.chip_buttons = [Button(start + i*chip_spacing, 550, chip_w, chip_w, str(v), GOLD if v==100 else WHITE, True) for i,v in enumerate(CHIP_VALUES)]

    def reset_game(self):
        self.deck = Deck(); self.player_hands = [[]]; self.dealer_hand = []; self.current_hand = 0; self.game_state = 'betting'
        # reinicia mazo, manos, mano actual y estado del juego
        self.money = 1000; self.current_bet = 0; self.bets = [0]; self.insurance_bet = 0  # dinero inicial y apuestas
        self.message = 'Selecciona fichas para apostar'; self.can_split = False; self.can_double = True; self.can_insurance = False
        # mensaje inicial 
    def reset_round(self):
        self.deck = Deck(); self.player_hands = [[]]; self.dealer_hand = []; self.current_hand = 0
        # reinicia datos de la ronda (sin reiniciar estadísticas)
        self.game_state = 'betting'; self.current_bet = 0; self.bets = [0]; self.insurance_bet = 0; self.message = 'Selecciona fichas para apostar'
        # estado de apuesta inicial
        self.can_split = False; self.can_double = True; self.can_insurance = False  

    def add_bet(self, amount):
        if self.money >= amount: self.current_bet += amount; self.money -= amount; return True
        return False  # agrega apuesta si hay saldo suficiente, devuelve True/False

    def get_card_value(self, card):
        if card.value in ('J','Q','K'): return 10  # figura vale 10
        if card.value == 'A': return 11  # As inicialmente 11 (ajustado luego)
        return int(card.value)  # number cards convierte a int

    def calculate_hand(self, hand):
        v = 0; aces = 0  # suma y cuenta de ases
        for c in hand:
            if c.value in ('J','Q','K'): v += 10  # suma figuras
            elif c.value == 'A': aces += 1  # cuenta ases
            else: v += int(c.value)  # suma valores numéricos
        for _ in range(aces): v += 11 if v+11<=21 else 1  # ajusta valor de ases para no pasarse
        return v  # devuelve total de la mano

    def calculate_visible_hand(self, hand):
        return self.calculate_hand([c for c in hand if c.shown])  # calcula total considerando solo cartas mostradas

    def render_stats(self):
        fig,(ax1,ax2) = plt.subplots(2,1,figsize=(6,6)); fig.patch.set_alpha(0.0)
        # crea figura con dos subplots y fondo transparente
        data = self.stats.get('historial_dinero')  # obtiene historial de dinero
        xs,ys = zip(*data) if data else ([0],[self.money])  # separa ejes X/Y
        ax1.plot(xs,ys,color='tab:green'); ax1.set_title('Evolución del dinero'); ax1.grid(True)
        # dibuja la evolución del dinero
        labels = ['Apostado','Ganado','Perdido']; vals = [self.stats.get('dinero_apostado',0), self.stats.get('dinero_ganado',0), self.stats.get('dinero_perdido',0)]
        ax2.bar(labels, vals, color=['#4C9F70','#2E8B57','#C94C4C']); ax2.set_title('Resumen')
        # dibuja resumen de estadísticas en barras
        canvas = FigureCanvasAgg(fig); canvas.draw(); renderer = canvas.get_renderer(); w,h = canvas.get_width_height()
    
        if hasattr(renderer, 'tostring_rgb'):
            raw = renderer.tostring_rgb(); mode = 'RGB'  
        else:
            raw_argb = renderer.tostring_argb()  
            try:
                import numpy as np  
                arr = np.frombuffer(raw_argb, dtype=np.uint8).reshape((h,w,4)); raw = arr[:,:, [1,2,3]].tobytes()
            except Exception:
                ba = bytearray(raw_argb); rgb = bytearray()
                for i in range(0, len(ba), 4): rgb.extend(ba[i+1:i+4])
                raw = bytes(rgb)  
            mode = 'RGB'  
        plt.close(fig)  
        return pygame.image.frombuffer(raw, (w,h), mode)  

    def deal_card(self, hand, shown=True):
        c = self.deck.draw(); c.shown = shown; hand.append(c); return c  # saca carta del mazo, la añade a la mano y devuelve la carta

    def deal_initial_cards(self):
        self.bets[0] = self.current_bet  # mueve la apuesta actual a la lista de apuestas
        self.deal_card(self.player_hands[0], True); self.deal_card(self.dealer_hand, True)
        self.deal_card(self.player_hands[0], True); self.deal_card(self.dealer_hand, False)
        # reparte dos cartas al jugador (mostradas) y dos al dealer (una oculta)
        self.game_state = 'playing'; self.check_split(); self.can_double = True  # actualiza estado y flags
        if self.dealer_hand and self.dealer_hand[0].value == 'A':
            self.can_insurance = True; self.message = 'As del dealer: ¿Tomar seguro? (Mitad de apuesta, paga 2:1)'
            # activa posibilidad de seguro si la carta visible del dealer es un As
        else:
            self.can_insurance = False  # de lo contrario no hay seguro
        if self.calculate_hand(self.player_hands[0]) == 21:
            if len(self.dealer_hand) > 1: self.dealer_hand[1].shown = True  # si jugador tiene blackjack, muestra carta oculta del dealer
            d = self.calculate_hand(self.dealer_hand); bet = self.bets[0]  # calcula manos y apuesta
            if d == 21: self.money += bet; self.message = 'Empate: ambos Blackjack'  # empate si dealer también tiene 21
            else: self.money += int(bet*2.5); self.message = 'Blackjack! Ganas 3:2.'  # paga 3:2 al jugador
            self.game_state = 'game_over'  # termina la ronda por blackjack

    def hit(self):
        if self.game_state != 'playing': return  # solo permite hit si está jugando
        hand = self.player_hands[self.current_hand]; self.deal_card(hand, True)  # reparte carta a la mano actual
        if self.calculate_hand(hand) > 21: self.next_hand()  # si se pasa, pasa a la siguiente mano

    def stand(self):
        self.next_hand()  # plantarse avanza a la siguiente mano / juego del dealer

    def split_hand(self):
        if not self.can_split or self.money < self.bets[self.current_hand]: return  # valida split
        hand = self.player_hands[self.current_hand]; card = hand.pop(); new = [card]; self.player_hands.insert(self.current_hand+1, new)
        # separa la carta en una nueva mano
        self.deal_card(hand, True); self.deal_card(new, True); bet = self.bets[self.current_hand]; self.money -= bet; self.bets.insert(self.current_hand+1, bet); self.check_split()
        # reparte segunda carta a cada mano, ajusta apuestas y dinero

    def double_down(self):
        if not self.can_double or self.money < self.bets[self.current_hand]: return  # valida doble
        self.money -= self.bets[self.current_hand]; self.bets[self.current_hand] *= 2  # duplica apuesta y descuenta dinero
        hand = self.player_hands[self.current_hand]; self.deal_card(hand, True); self.next_hand()  # reparte una carta y avanza

    def check_split(self):
        hand = self.player_hands[self.current_hand]  # obtiene la mano actual
        if len(hand) == 2:
            self.can_split = (self.get_card_value(hand[0]) == self.get_card_value(hand[1]) and self.money >= self.bets[self.current_hand])
            # permite split si ambos valores coinciden y hay dinero suficiente
        else:
            self.can_split = False  # no permite split si no hay exactamente 2 cartas

    def next_hand(self):
        self.current_hand += 1  # avanza índice de mano actual
        if self.current_hand >= len(self.player_hands): self.play_dealer_hand()  # si ya no hay manos de jugador, juega el dealer
        else: self.message = f'Jugando mano {self.current_hand+1}'; self.check_split(); self.can_double = (len(self.player_hands[self.current_hand]) == 2)
        # si hay otra mano, actualiza mensaje y flags

    def play_dealer_hand(self):
        if len(self.dealer_hand) > 1: self.dealer_hand[1].shown = True  # muestra la carta oculta del dealer
        if self.insurance_bet > 0 and self.calculate_hand(self.dealer_hand) == 21: self.money += self.insurance_bet * 3; self.message = 'Seguro ganado! (2:1)'
        # resuelve seguro si corresponde
        while self.calculate_hand(self.dealer_hand) < 17: self.deal_card(self.dealer_hand, True)  # dealer roba hasta 17
        self.settle_bets()  # liquida apuestas al final

    def settle_bets(self):
        dealer = self.calculate_hand(self.dealer_hand); msgs = []; total_ganado = 0; total_perdido = 0
        # calcula mano del dealer y prepara contadores
        for i, hand in enumerate(self.player_hands):
            p = self.calculate_hand(hand); bet = self.bets[i]; self.stats['dinero_apostado'] += bet
            # por cada mano de jugador calcula resultado y actualiza estadísticas
            if p > 21: msgs.append(f'Mano {i+1}: Te pasaste'); total_perdido += bet
            elif dealer > 21 or p > dealer: self.money += bet * 2; msgs.append(f'Mano {i+1}: Ganas'); total_ganado += bet
            elif p < dealer: msgs.append(f'Mano {i+1}: Pierdes'); total_perdido += bet
            else: self.money += bet; msgs.append(f'Mano {i+1}: Empate')
        self.stats['manos_jugadas'] += len(self.player_hands); self.stats['dinero_ganado'] += total_ganado; self.stats['dinero_perdido'] += total_perdido
        # actualiza estadísticas acumuladas
        self.stats['historial_dinero'].append((len(self.stats['historial_dinero']), self.money))  # añade punto al historial
        self.message = ' | '.join(msgs); self.game_state = 'game_over'  # prepara mensaje final y marca fin de ronda

    def draw(self, surf):
        surf.fill(GREEN)  # llena fondo con color mesa
        for i, card in enumerate(self.dealer_hand): card.draw(surf, (350 + i*90, 100))  # dibuja cartas del dealer
        if self.dealer_hand:
            vis = self.calculate_visible_hand(self.dealer_hand); draw_bubble(surf, 350, 55, f'Dealer: {vis}')
            # muestra total visible del dealer
            if self.can_insurance and self.game_state == 'playing':
                self.insurance_button.draw(surf); bx = self.insurance_button.rect.x; by = self.insurance_button.rect.y
                draw_bubble(surf, bx, by - 42, '¿Tomar seguro? (2:1)')  # botón y burbuja de seguro si aplica
        if getattr(self, 'show_stats', False):
            stats_surf = self.render_stats(); x = W//2 - stats_surf.get_width()//2; y = H//2 - stats_surf.get_height()//2; surf.blit(stats_surf, (x, y))
            # muestra la superficie de estadísticas centrada
            close_rect = pygame.Rect(x + stats_surf.get_width() - 30, y, 30, 30); pygame.draw.rect(surf, RED, close_rect)
            txt = font.render('X', True, WHITE); surf.blit(txt, txt.get_rect(center=close_rect.center)); return
            # dibuja botón de cerrar y retorna (no dibuja el resto)
        for hi, hand in enumerate(self.player_hands):
            for i, card in enumerate(hand): card.draw(surf, (350 + i*90, 400 + hi*150))  # dibuja cartas de cada mano de jugador
            if hi == self.current_hand and self.game_state == 'playing' and len(self.player_hands) > 1: pygame.draw.rect(surf, GOLD, (340, 390 + hi*150, 400, 140), 2)
            # resalta la mano actual si hay varias manos
            if hi == self.current_hand: draw_bubble(surf, W - 180, 420 + hi*150, f'Total: {self.calculate_hand(hand)}')  # muestra total de la mano actual
        if self.game_state == 'playing':
            self.hit_button.draw(surf); self.stand_button.draw(surf)  # dibuja botones Hit y Stand
            self.split_button.enabled = self.can_split; self.split_button.draw(surf)  # actualiza y dibuja Split
            self.double_button.enabled = self.can_double; self.double_button.draw(surf)  # actualiza y dibuja Double
        else:
            self.deal_button.enabled = self.current_bet > 0; self.deal_button.draw(surf); [b.draw(surf) for b in self.chip_buttons]
            # si no está jugando, dibuja Deal y fichas para apostar
        if self.money == 0 and self.game_state != 'playing':
            self.reset_button.draw(surf); draw_bubble(surf, W//2 - 200, self.reset_button.rect.y - 60, '¡Te quedaste sin dinero! Presiona Reset para empezar de nuevo')
            # muestra mensaje y botón Reset si no hay dinero
        self.stats_button.draw(surf); surf.blit(font.render(f'Dinero: ${self.money}', True, WHITE), (50,50)); surf.blit(font.render(f'Apuesta: ${self.current_bet}', True, WHITE), (50,90)); surf.blit(font.render(self.message, True, WHITE), (W//2 - 150, 300))
        # dibuja botón de estadísticas y texto con dinero, apuesta y mensaje


game = Game(); clock = pygame.time.Clock(); running = True  # inicializa juego, reloj y flag de bucle principal
# Bucle principal del juego
while running:
    for event in pygame.event.get():
        if event.type == pygame.QUIT: running = False  # cierra si se solicita cerrar la ventana
        elif event.type == pygame.MOUSEBUTTONDOWN:
            mx, my = pygame.mouse.get_pos()  # obtiene posición del clic
            if getattr(game, 'show_stats', False):
                stats_surf = game.render_stats(); x = W//2 - stats_surf.get_width()//2; y = H//2 - stats_surf.get_height()//2; close_rect = pygame.Rect(x + stats_surf.get_width() - 30, y, 30, 30)
                if close_rect.collidepoint((mx,my)): game.show_stats = False  # cierra la vista de estadísticas si se hace clic en la X
                continue  # ignora el resto de interacciones mientras se muestran estadísticas
            if game.game_state == 'playing':
                if game.insurance_button.rect.collidepoint((mx,my)) and game.can_insurance:
                    insurance_amount = game.bets[0] // 2
                    if game.money >= insurance_amount: game.money -= insurance_amount; game.insurance_bet = insurance_amount; game.can_insurance = False; game.message = 'Seguro tomado'
                    # procesa la toma de seguro si se hace clic y hay dinero suficiente
                elif game.hit_button.rect.collidepoint((mx,my)): game.hit()  # clic en Hit
                elif game.stand_button.rect.collidepoint((mx,my)): game.stand()  # clic en Stand
                elif game.split_button.rect.collidepoint((mx,my)) and game.can_split: game.split_hand()  # clic en Split si permitido
                elif game.double_button.rect.collidepoint((mx,my)) and game.can_double: game.double_down()  # clic en Double si permitido
            else:
                if game.reset_button.rect.collidepoint((mx,my)) and game.money == 0: game.reset_game()  # reset si no hay dinero
                elif game.stats_button.rect.collidepoint((mx,my)): game.show_stats = True  # muestra estadísticas
                elif game.deal_button.rect.collidepoint((mx,my)) and game.current_bet > 0:
                    if game.game_state == 'game_over': game.reset_round()
                    else: game.deal_initial_cards()  # inicia reparto si se clickea Deal y hay apuesta
                else:
                    for b in game.chip_buttons:
                        if b.rect.collidepoint((mx,my)): v = int(b.text); game.add_bet(v); break
                        # añade apuesta si se clickea alguna ficha
    game.draw(screen); pygame.display.flip(); clock.tick(60)  # dibuja pantalla, actualiza display y controla FPS
pygame.quit(); sys.exit()  # finaliza pygame y sale del programa
