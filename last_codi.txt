import sys
import math
import heapq

class Command:
    def __init__(self):
        self.commands = []

    def wait(self):
        self.commands.append("WAIT")

    def message(self, message):
        self.commands.append("MESSAGE" + " " + str(message))
    
    def line(self, sourceId, targetId, strength):
        self.commands.append("LINE" + " " + str(sourceId) + " " + str(targetId) + " " + str(strength))
    
    def BEACON(self, targetId, strength):
        self.commands.append("BEACON" + " " + str(targetId) + " " + str(strength))
    
    def log(self, message):
        print(message, file=sys.stderr, flush=True)
    
    def execute(self):
        print(';'.join(self.commands))
        self.commands = []

class Cell:
    def __init__(self, type, initial_resources, neigh_0, neigh_1, neigh_2, neigh_3, neigh_4, neigh_5):
        self._type = type
        self._initial_resources = initial_resources
        self.arrayNeightboor = [n for n in (neigh_0, neigh_1, neigh_2, neigh_3, neigh_4, neigh_5) if n != -1]
        self.strenght = 0 # for the calcule 
        self.isBeacon = False # add new attribut for calcul 
    
    def update_value(self, resources, my_ants, opp_ants):
        self._ressources = resources
        self._my_ants = my_ants
        self._opp_ants = opp_ants
    
    def print_value(self, number_of_cells):
        message = f"-type = {str(self._type)}// -initial_resources = {str(self._initial_resources)}//-arrayneightboor = {str(self.arrayNeightboor)}//-ressource = {str(self._ressources)}// -my_ants = {str(self._my_ants)}// -opp_ants = {str(self._opp_ants)}"
        return message
    
    def reset_data_Not_persistante(self):
        self.strenght = 0
        self.isBeacon = False #false == not beacon

class Board:
    def __init__(self):
        self.number_of_cells = int(input())  # amount of hexagonal cells in this map
        self.arrayCell = []
        self.arrayMyBaseIndex = []
        self.arrayOppBaseIndex = []
        self.command = Command()
        self.bfs_trees = {}
        self.nb_fourmis_initial = 0
        self.lock = False
        for i in range(self.number_of_cells):
            # _type: 0 for empty, 1 for eggs, 2 for crystal
            # initial_resources: the initial amount of eggs/crystals on this cell
            # neigh_0: the index of the neighbouring cell for each direction
            _type, initial_resources, neigh_0, neigh_1, neigh_2, neigh_3, neigh_4, neigh_5 = [int(j) for j in input().split()]
            self.arrayCell.append(Cell(_type, initial_resources, neigh_0, neigh_1, neigh_2, neigh_3, neigh_4, neigh_5))
        self.number_of_bases = int(input())
        for i in input().split():
            self.arrayMyBaseIndex.append(int(i))
        for i in input().split():
            self.arrayOppBaseIndex.append(int(i))

    def update_value_gen(self, _nb_fourmis_initial):
        self.nb_fourmis_initial = _nb_fourmis_initial

    def bfs(self, start):
        if start in self.bfs_trees:
            return self.bfs_trees[start]

        visited = [False]*self.number_of_cells
        queue = [start]
        visited[start] = True
        parent = {start: None}

        while queue:
            current_cell = queue.pop(0)
            for neighbour in self.arrayCell[current_cell].arrayNeightboor:
                if not visited[neighbour]:
                    queue.append(neighbour)
                    visited[neighbour] = True
                    parent[neighbour] = current_cell

        self.bfs_trees[start] = parent  # store the bfs result
        return parent
    
    def shortest_path(self, start, end):
        parent = self.bfs(start)
        path = [end]
        while path[-1] != start:
            path.append(parent[path[-1]])
        path.reverse()
        return path
    
    #----------------------------2 eme methode calcule chemin plus court avec amelioration pour passer par des chemin plus optimaux
    def get_weighted_distance(self, cell_1, cell_2):
        distance = 1  # default distance between two cells is 1
        egg_weight = 0.4  # this can be adjusted based on how much you want to prioritize egg cells
        crystal_weight = 0.6  # this can be adjusted based on how much you want to prioritize crystal cells

        # If either cell has resources, reduce the distance based on the type and amount of resources
        if self.arrayCell[cell_1]._ressources > 0:
            if self.arrayCell[cell_1]._type == 1:  # eggs
                distance /= (1 + egg_weight * self.arrayCell[cell_1]._ressources)
            elif self.arrayCell[cell_1]._type == 2:  # crystals
                distance /= (1 + crystal_weight * self.arrayCell[cell_1]._ressources)

        if self.arrayCell[cell_2]._ressources > 0:
            if self.arrayCell[cell_2]._type == 1:  # eggs
                distance /= (1 + egg_weight * self.arrayCell[cell_2]._ressources)
            elif self.arrayCell[cell_2]._type == 2:  # crystals
                distance /= (1 + crystal_weight * self.arrayCell[cell_2]._ressources)

        #self.command.log(distance)
        return distance
    
    def bfs_2(self, start):
        if start in self.bfs_trees:
            return self.bfs_trees[start]

        visited = [False]*self.number_of_cells
        queue = [(0, start)]  # The queue now stores tuples of (cost, cell)
        visited[start] = True
        parent = {start: None}
        costs = {start: 0}

        while queue:
            queue.sort()  # Make sure the queue is sorted by cost
            current_cost, current_cell = heapq.heappop(queue)  # Use a heap to efficiently get the smallest cost

            for neighbour in self.arrayCell[current_cell].arrayNeightboor:
                new_cost = current_cost + self.get_weighted_distance(current_cell, neighbour)
                if not visited[neighbour] or new_cost < costs[neighbour]:
                    costs[neighbour] = new_cost
                    parent[neighbour] = current_cell
                    heapq.heappush(queue, (new_cost, neighbour))
                    visited[neighbour] = True

        self.bfs_trees[start] = parent  # store the bfs result
        return parent
    
    def shortest_path_2(self, start, end):
        parent = self.bfs_2(start)
        path = [end]
        while path[-1] != start:
            path.append(parent[path[-1]])
        path.reverse()
        return path
    #----------------------------------------------------------------------------------------------------------------------------------
    
    def get_resource_cells(self):
        return [i for i in range(self.number_of_cells) if self.arrayCell[i]._ressources > 0]
    
    #Dans cette méthode, pour chaque cellule avec des fourmis alliées, elle trouve le chemin le plus court vers chaque cellule avec des ressources, puis trie ces chemins par longueur en ordre croissant.
    #un dictionnaire où chaque clé est l'indice d'une cellule de fourmi et chaque valeur est une liste de tuples. Chaque tuple contient l'indice d'une cellule de ressource et le chemin le plus court vers cette ressource, triés par longueur de chemin en ordre croissant.
    def dictionnaire_fourmis_allie_and_each_ressource_shortest_path(self):
        resource_cells = self.get_resource_cells()
        shortest_paths = {}

        for i in range(self.number_of_cells):
            if self.arrayCell[i]._my_ants > 0:
                paths = [(j, self.shortest_path(i, j)) for j in resource_cells]
                # Sort the paths by length in ascending order
                paths.sort(key=lambda x: len(x[1]))
                shortest_paths[i] = paths

        return shortest_paths
    
    # idem que foonction juste en haut mais depuis une case spécifique
    def dictionnaire_Mybase_and_each_ressource_shortest_path(self, is_reverse):
        resource_cells = self.get_resource_cells()
        shortest_paths = {}

        for i in self.arrayMyBaseIndex:
            paths = [(j, self.shortest_path_2(i, j)) for j in resource_cells]
            # Sort the paths by length in ascending order
            if is_reverse == True:
                paths.sort(key=lambda x: len(x[1]),reverse=True)
            else:
                paths.sort(key=lambda x: len(x[1]))
            shortest_paths[i] = paths

        return shortest_paths
    
    def get_total_ants_(self,allie_ou_pas):
        total = 0
        if allie_ou_pas:
            for i in self.arrayCell:
                if i._my_ants > 0:
                    total += i._my_ants
        else:
            for i in self.arrayCell:
                if i._opp_ants > 0:
                    total += i._opp_ants
        return total
    
    def total_spot_ressource_oeuf(self):
        total = 0
        for i in self.arrayCell:
            if (i._type == 1) and i._ressources > 0:
                total+=1
        return total
    
    def total_spot_ressource_cristal(self):
        total = 0
        for i in self.arrayCell:
            if (i._type == 2) and i._ressources > 0:
                total+=1
        return total
        
    def play_2(self):

        if (self.total_spot_ressource_cristal() + self.total_spot_ressource_oeuf()) > 10:
            self.command.message("Great map")
            #Recuperation nombre oeuf map et definir nombre oeuf a prendre pour atteindre quota
            if self.lock == False:
                self.nb_fourmis_initial = self.get_total_ants_(True)
                self.lock = True
            Pourcent_limit = 70#100
            nb_oeuf_map = 0
            nb_fourmis_now = 0
            nb_oeuf_now = 0
            quota = 0
            for i in self.arrayCell:
                if i._type == 1:
                    nb_oeuf_now += i._ressources
            for i in self.arrayCell:
                if i._type == 1:
                    nb_oeuf_map += i._initial_resources
            for i in self.arrayCell:
                if i._my_ants > 0:
                    nb_fourmis_now += i._my_ants
            nb_fourmis_get = nb_fourmis_now - self.nb_fourmis_initial
            quota = int((100 * nb_fourmis_get)/nb_oeuf_map)
            if nb_oeuf_now == 0:
                quota = Pourcent_limit + 1
            #calcule array oeuf    
            if (self.total_spot_ressource_cristal() + self.total_spot_ressource_oeuf()) > 10:
                self.command.message("loin")
                #recuperation chemin plus proche avec tous les oeufs de la map
                dict_indexPath_oeufs_map = {}
                for cle, value in self.dictionnaire_Mybase_and_each_ressource_shortest_path(False).items():#loin
                    array_path = []
                    for elem in value:
                        if self.arrayCell[elem[0]]._type == 1:
                            array_path.append(elem)
                    dict_indexPath_oeufs_map[cle] = array_path
            else:
                self.command.message("pres")
                #recuperation chemin plus proche avec tous les oeufs de la map
                dict_indexPath_oeufs_map = {}
                for cle, value in self.dictionnaire_Mybase_and_each_ressource_shortest_path(False).items():#pres
                    array_path = []
                    for elem in value:
                        if self.arrayCell[elem[0]]._type == 1:
                            array_path.append(elem)
                    dict_indexPath_oeufs_map[cle] = array_path
            
            if quota < Pourcent_limit:
                #parcours tous les oeuf et je vais chercher
                array_index_parcouru = []
                array_index_miss = []
                nb_fourmis_restant = self.get_total_ants_(True)
                tour = 0
                while(True):
                    tour+=1
                    self.command.log("--------------------")
                    self.command.log(tour)
                    self.command.log(self.total_spot_ressource_oeuf())
                    #self.command.log(self.total_spot_ressource_cristal())
                    self.command.log(array_index_parcouru)
                    self.command.log(array_index_miss)
                    if self.total_spot_ressource_oeuf() == len(array_index_parcouru):
                        break
                    for cle, value in dict_indexPath_oeufs_map.items():
                        for index, path in value:
                            is_find = False
                            for k in array_index_parcouru:
                                if k == index:
                                    is_find = True
                            if is_find == False:
                                array_index_parcouru.append(index)
                            is_find_2 = False
                            for j in array_index_miss:
                                if j == index:
                                    is_find_2 = True
                                    break
                            if is_find_2 == True:
                                continue
                            nb_ressource = self.arrayCell[index]._ressources
                            nb_fourmis_eachCell = nb_fourmis_restant//len(path)
                            if nb_fourmis_eachCell != 0:
                                if nb_fourmis_eachCell > nb_ressource:
                                    too_many_ants = nb_fourmis_eachCell - nb_ressource
                                    nb_fourmis_eachCell = nb_fourmis_eachCell - too_many_ants
                                    if self.arrayCell[index]._opp_ants > (nb_fourmis_eachCell + too_many_ants):
                                        continue
                                if self.arrayCell[index]._opp_ants > nb_fourmis_eachCell:
                                    continue
                                poid = (nb_fourmis_eachCell *100)//self.get_total_ants_(True)
                                nb_fourmis_restant = nb_fourmis_restant - (nb_fourmis_eachCell * len(path))
                                for i in path:
                                    self.arrayCell[i].isBeacon = True
                                    self.arrayCell[i].strength = poid
                                array_index_miss.append(index)
                                break
            else:
                #parcours tous les oeuf et cristaux et je vais chercher
                array_index_parcouru = []
                array_index_miss = []
                nb_fourmis_restant = self.get_total_ants_(True)
                tour = 0
                while(True):
                    tour+=1
                    self.command.log("--------------------")
                    self.command.log(tour)
                    self.command.log(self.total_spot_ressource_oeuf())
                    #self.command.log(self.total_spot_ressource_cristal())
                    self.command.log(array_index_parcouru)
                    self.command.log(array_index_miss)
                    if (self.total_spot_ressource_oeuf() + self.total_spot_ressource_cristal()) == len(array_index_parcouru):
                        break
                    for cle, value in self.dictionnaire_Mybase_and_each_ressource_shortest_path(False).items():#plus loin
                        for index, path in value:
                            is_find = False
                            for k in array_index_parcouru:
                                if k == index:
                                    is_find = True
                            if is_find == False:
                                array_index_parcouru.append(index)
                            is_find_2 = False
                            for j in array_index_miss:
                                if j == index:
                                    is_find_2 = True
                                    break
                            if is_find_2 == True:
                                continue
                            nb_ressource = self.arrayCell[index]._ressources
                            nb_fourmis_eachCell = nb_fourmis_restant//len(path)
                            if nb_fourmis_eachCell != 0:
                                if nb_fourmis_eachCell > nb_ressource:
                                    too_many_ants = nb_fourmis_eachCell - nb_ressource
                                    nb_fourmis_eachCell = nb_fourmis_eachCell - too_many_ants
                                    if self.arrayCell[index]._opp_ants > (nb_fourmis_eachCell + too_many_ants):
                                        continue
                                if self.arrayCell[index]._opp_ants > nb_fourmis_eachCell:
                                    continue
                                poid = (nb_fourmis_eachCell *100)//self.get_total_ants_(True)
                                nb_fourmis_restant = nb_fourmis_restant - (nb_fourmis_eachCell * len(path))
                                for i in path:
                                    self.arrayCell[i].isBeacon = True
                                    self.arrayCell[i].strength = poid
                                array_index_miss.append(index)
                                break
        else:
            if self.total_spot_ressource_cristal() > 0:
                self.command.message("encore cristaud")
                #parcours tous cristaux et je vais chercher
                array_index_parcouru = []
                array_index_miss = []
                nb_fourmis_restant = self.get_total_ants_(True)
                tour = 0
                while(True):
                    tour+=1
                    self.command.log("--------------------")
                    self.command.log(tour)
                    self.command.log(self.total_spot_ressource_oeuf())
                    self.command.log(self.total_spot_ressource_cristal())
                    self.command.log(array_index_parcouru)
                    self.command.log(array_index_miss)
                    if self.total_spot_ressource_cristal() == len(array_index_parcouru):
                        break
                    for cle, value in self.dictionnaire_Mybase_and_each_ressource_shortest_path(False).items():#plus pres
                        for index, path in value:
                            if self.arrayCell[index]._type == 1:
                                continue
                            is_find = False
                            for k in array_index_parcouru:
                                if k == index:
                                    is_find = True
                            if is_find == False:
                                array_index_parcouru.append(index)
                            is_find_2 = False
                            for j in array_index_miss:
                                if j == index:
                                    is_find_2 = True
                                    break
                            if is_find_2 == True:
                                continue
                            nb_ressource = self.arrayCell[index]._ressources
                            nb_fourmis_eachCell = nb_fourmis_restant//len(path)
                            if nb_fourmis_eachCell != 0:
                                if nb_fourmis_eachCell > nb_ressource:
                                    too_many_ants = nb_fourmis_eachCell - nb_ressource
                                    nb_fourmis_eachCell = nb_fourmis_eachCell - too_many_ants
                                    if self.arrayCell[index]._opp_ants > (nb_fourmis_eachCell + too_many_ants):
                                        continue
                                if self.arrayCell[index]._opp_ants > nb_fourmis_eachCell:
                                    continue
                                poid = (nb_fourmis_eachCell *100)//self.get_total_ants_(True)
                                nb_fourmis_restant = nb_fourmis_restant - (nb_fourmis_eachCell * len(path))
                                for i in path:
                                    self.arrayCell[i].isBeacon = True
                                    self.arrayCell[i].strength = poid
                                array_index_miss.append(index)
                                break
            else:
                #parcours tous les oeuf et cristaux restant et je vais chercher
                self.command.message("plus cristaud")
                array_index_parcouru = []
                array_index_miss = []
                nb_fourmis_restant = self.get_total_ants_(True)
                tour = 0
                while(True):
                    tour+=1
                    self.command.log("--------------------")
                    self.command.log(tour)
                    self.command.log(self.total_spot_ressource_oeuf())
                    #self.command.log(self.total_spot_ressource_cristal())
                    self.command.log(array_index_parcouru)
                    self.command.log(array_index_miss)
                    if (self.total_spot_ressource_oeuf() + self.total_spot_ressource_cristal()) == len(array_index_parcouru):
                        break
                    for cle, value in self.dictionnaire_Mybase_and_each_ressource_shortest_path(False).items():#plus pres
                        for index, path in value:
                            is_find = False
                            for k in array_index_parcouru:
                                if k == index:
                                    is_find = True
                            if is_find == False:
                                array_index_parcouru.append(index)
                            is_find_2 = False
                            for j in array_index_miss:
                                if j == index:
                                    is_find_2 = True
                                    break
                            if is_find_2 == True:
                                continue
                            nb_ressource = self.arrayCell[index]._ressources
                            nb_fourmis_eachCell = nb_fourmis_restant//len(path)
                            if nb_fourmis_eachCell != 0:
                                if nb_fourmis_eachCell > nb_ressource:
                                    too_many_ants = nb_fourmis_eachCell - nb_ressource
                                    nb_fourmis_eachCell = nb_fourmis_eachCell - too_many_ants
                                    if self.arrayCell[index]._opp_ants > (nb_fourmis_eachCell + too_many_ants):
                                        continue
                                if self.arrayCell[index]._opp_ants > nb_fourmis_eachCell:
                                    continue
                                poid = (nb_fourmis_eachCell *100)//self.get_total_ants_(True)
                                nb_fourmis_restant = nb_fourmis_restant - (nb_fourmis_eachCell * len(path))
                                for i in path:
                                    self.arrayCell[i].isBeacon = True
                                    self.arrayCell[i].strength = poid
                                array_index_miss.append(index)
                                break





        # Mise en place des action
        do_wait = False
        for i in range(self.number_of_cells):
            if self.arrayCell[i].isBeacon == True:
                do_wait = True
                self.command.BEACON(i, self.arrayCell[i].strength)
        if do_wait == False:
            self.command.wait()
    
    def play(self):
        """shortest_paths = self.dictionnaire_fourmis_allie_and_each_ressource_shortest_path()
        for ant_cell_index, paths in shortest_paths.items():
            message = f"Ant cell {ant_cell_index} shortest paths:"
            self.command.log(message)
            for resource_index, path in paths:
               message = f"To resource cell {resource_index}: {path}"
               self.command.log(message)


        for i in range(self.number_of_cells):
            message = f"case {str(i)}\n{str(self.arrayCell[i].print_value(self.number_of_cells))}"
            self.command.log(message)"""
        
        """shortest_paths = self.dictionnaire_fourmis_allie_and_each_ressource_shortest_path()
        chemin_proche = [shortest_paths[ant_cell_index][0][0] for ant_cell_index, paths in shortest_paths.items()]
        chemin_proche_not_doublon = list(set(chemin_proche))
        nombre_fouris_map_allie = self.get_total_ants_(1)
        self.command.log(nombre_fouris_map_allie)
        for i in chemin_proche_not_doublon:
            path = self.shortest_path(self.arrayMyBaseIndex[0], i)
            repartion = nombre_fouris_map_allie/len(path)
            if int(repartion) > self.arrayCell[i]._ressources:"""
        #test = self.dictionnaire_Mybase_and_each_ressource_shortest_path()
        #self.command.log(test)
        # Quota de fourmis par base
        nb_fourmisEachBase = self.get_total_ants_(True)//len(self.arrayMyBaseIndex)
        nb_fourmi_restant = self.get_total_ants_(True) % len(self.arrayMyBaseIndex)
        quota_eachBaseAnts = (nb_fourmisEachBase*100)/self.get_total_ants_(True)
        #Recuperation nombre oeuf map et definir nombre oeuf a prendre pour atteindre quota
        if self.lock == False:
            self.nb_fourmis_initial = self.get_total_ants_(True)
            self.lock = True
        Pourcent_limit = 100#100
        nb_oeuf_map = 0
        nb_fourmis_now = 0
        nb_oeuf_now = 0
        quota = 0
        for i in self.arrayCell:
            if i._type == 1:
                nb_oeuf_now += i._ressources
        for i in self.arrayCell:
            if i._type == 1:
                nb_oeuf_map += i._initial_resources
        for i in self.arrayCell:
            if i._my_ants > 0:
                nb_fourmis_now += i._my_ants
        nb_fourmis_get = nb_fourmis_now - self.nb_fourmis_initial
        quota = int((100 * nb_fourmis_get)/nb_oeuf_map)
        if nb_oeuf_now == 0:
            quota = Pourcent_limit + 1
        # calcule pourcent des cristaud pris
        nb_cristaud_base = 0
        nb_cristaud_now = 0
        pourcent_cristaud = 80
        for i in self.arrayCell:
            if i._type == 2:
                nb_cristaud_base += i._initial_resources
        for i in self.arrayCell:
            if i._type == 2:
                nb_cristaud_now += i._ressources
        self.command.log(nb_cristaud_now)
        cristaux_quota = int((nb_cristaud_now * 100)/nb_cristaud_base)
        if cristaux_quota < pourcent_cristaud:
            quota = Pourcent_limit + 1
        #recuperation chemin plus proche avec tous les oeufs de la map
        dict_indexPath_oeufs_map = {}
        for cle, value in self.dictionnaire_Mybase_and_each_ressource_shortest_path(False).items():
            array_path = []
            for elem in value:
                if self.arrayCell[elem[0]]._type == 1:
                    array_path.append(elem)
            dict_indexPath_oeufs_map[cle] = array_path
        #parcours chacun des oeufs les plus proche pour attribuer des fourmis
        array_cell_remove = []
        message = f"quota : {quota} // nb_oeuf_now : {nb_oeuf_now}"
        self.command.log(message)
        if quota < Pourcent_limit:
            self.command.message("quota inf")
            for cle, value in dict_indexPath_oeufs_map.items():
                nb_fourmis_restant = nb_fourmisEachBase
                for index, path in value:
                    save_fourmis_restant = nb_fourmis_restant
                    distance = len(path)
                    repartion_ants = nb_fourmis_restant//distance
                    for remove in array_cell_remove:
                        if remove == path[-1]:
                            repartion_ants = 0
                    if repartion_ants != 0:
                        #break
                        #continue
                        message = f"nb fourmis tot : {self.get_total_ants_(True)} - nb_fourmis_restant : {nb_fourmis_restant} - repartion_ants : {repartion_ants} - self.arrayCell[index]._ressources : {self.arrayCell[index]._ressources}"
                        self.command.log(message)
                        if repartion_ants > self.arrayCell[index]._ressources:
                            too_many_ants = repartion_ants - self.arrayCell[index]._ressources
                            repartion_ants = repartion_ants - too_many_ants
                            self.command.log("Much ants!")
                            self.command.log(repartion_ants)
                        nb_fourmis_restant -= (repartion_ants *distance)
                        strength = (repartion_ants*100)//self.get_total_ants_(True)
                        #verif si une balise etait pas deja présente et reajuster si besoin
                        new_strength = strength
                        cell_remove = []
                        for i in path:
                            if self.arrayCell[i].isBeacon == True:
                                if self.arrayCell[i].strength > new_strength:
                                    new_repartion_ants = save_fourmis_restant//(distance -1)
                                    if new_repartion_ants != 0:
                                        if new_repartion_ants > self.arrayCell[index]._ressources:
                                            too_many_ants = new_repartion_ants - self.arrayCell[index]._ressources
                                            new_repartion_ants = new_repartion_ants - too_many_ants
                                        save_fourmis_restant -= (new_repartion_ants *(distance-1))
                                        new_strength = (new_repartion_ants*100)//self.get_total_ants_(True)
                                        cell_remove.append(i)
                                else:
                                    diff_strength = new_strength - self.arrayCell[i].strength
                                    self.arrayCell[i].strength += diff_strength
                        # mise a jour des poids sur chaque case
                        for element in path:
                            find = False
                            for remove_elem in cell_remove:
                                if remove_elem == element:
                                    find = True
                                    break
                            if find == False:
                                self.arrayCell[element].isBeacon = True
                                self.arrayCell[element].strength = new_strength
                        #mise a jour des cibles deja prise
                        if self.arrayCell[path[-1]].isBeacon == True:
                            array_cell_remove.append(path[-1])
                        #mise a jour du nombre de fourmis
                        nb_fourmis_restant = save_fourmis_restant
                        """for i in range(distance):
                            self.command.BEACON(path[i], strength)"""
                        self.command.log(f"nb fourmis par base {nb_fourmis_restant} // base {cle} // target {index} // poid cases {strength}")
                        if nb_fourmis_restant == 0:
                            break
        else:
            self.command.message("quota>")
            # aller chercher les cristaud
            for cle, value in self.dictionnaire_Mybase_and_each_ressource_shortest_path(False).items():
                nb_fourmis_restant = nb_fourmisEachBase
                for index, path in value:
                    """message = f"self.get_total_ants_(False) : {self.get_total_ants_(False)} // self.get_total_ants_(True) {self.get_total_ants_(True)}"
                    self.command.log(message)
                    if self.get_total_ants_(False) <= self.get_total_ants_(True):"""
                    if self.arrayCell[index]._type == 1:
                        continue
                    save_fourmis_restant = nb_fourmis_restant
                    distance = len(path)
                    repartion_ants = nb_fourmis_restant//distance
                    for remove in array_cell_remove:
                        if remove == path[-1]:
                            repartion_ants = 0
                    if repartion_ants != 0:
                        #break
                        #continue
                        message = f"nb fourmis tot : {self.get_total_ants_(True)} - nb_fourmis_restant : {nb_fourmis_restant} - repartion_ants : {repartion_ants} - self.arrayCell[index]._ressources : {self.arrayCell[index]._ressources}"
                        self.command.log(message)
                        if repartion_ants > self.arrayCell[index]._ressources:
                            too_many_ants = repartion_ants - self.arrayCell[index]._ressources
                            repartion_ants = repartion_ants - too_many_ants
                            self.command.log("Much ants!")
                            self.command.log(repartion_ants)
                        nb_fourmis_restant -= (repartion_ants *distance)
                        strength = (repartion_ants*100)//self.get_total_ants_(True)
                        #verif si une balise etait pas deja présente et reajuster si besoin
                        new_strength = strength
                        cell_remove = []
                        for i in path:
                            if self.arrayCell[i].isBeacon == True:
                                if self.arrayCell[i].strength > new_strength:
                                    new_repartion_ants = save_fourmis_restant//(distance -1)
                                    if new_repartion_ants != 0:
                                        if new_repartion_ants > self.arrayCell[index]._ressources:
                                            too_many_ants = new_repartion_ants - self.arrayCell[index]._ressources
                                            new_repartion_ants = new_repartion_ants - too_many_ants
                                        save_fourmis_restant -= (new_repartion_ants *(distance-1))
                                        new_strength = (new_repartion_ants*100)//self.get_total_ants_(True)
                                        cell_remove.append(i)
                                else:
                                    diff_strength = new_strength - self.arrayCell[i].strength
                                    self.arrayCell[i].strength += diff_strength
                        # mise a jour des poids sur chaque case
                        for element in path:
                            find = False
                            for remove_elem in cell_remove:
                                if remove_elem == element:
                                    find = True
                                    break
                            if find == False:
                                self.arrayCell[element].isBeacon = True
                                self.arrayCell[element].strength = new_strength
                        #mise a jour des cibles deja prise
                        if self.arrayCell[path[-1]].isBeacon == True:
                            array_cell_remove.append(path[-1])
                        #mise a jour du nombre de fourmis
                        nb_fourmis_restant = save_fourmis_restant
                        """for i in range(distance):
                            self.command.BEACON(path[i], strength)"""
                        self.command.log(f"nb fourmis par base {nb_fourmis_restant} // base {cle} // target {index} // poid cases {strength}")
                        if nb_fourmis_restant == 0:
                            break
        # Mise en place des action
        do_wait = False
        for i in range(self.number_of_cells):
            if self.arrayCell[i].isBeacon == True:
                do_wait = True
                self.command.BEACON(i, self.arrayCell[i].strength)
        if do_wait == False:
            self.command.wait()
    
    def loopGame(self):
        
        # game loop
        while True:
            for i in range(self.number_of_cells):
                # resources: the current amount of eggs/crystals on this cell
                # my_ants: the amount of your ants on this cell
                # opp_ants: the amount of opponent ants on this cell
                resources, my_ants, opp_ants = [int(j) for j in input().split()]
                self.arrayCell[i].update_value(resources, my_ants, opp_ants)
                self.arrayCell[i].reset_data_Not_persistante()

            # test
            #path = self.shortest_path_2(59, 17)
            #self.command.log(path)
            # To debug: print("Debug messages...", file=sys.stderr, flush=True)
            self.play_2()
            self.command.execute()
            # WAIT | LINE <sourceIdx> <targetIdx> <strength> | BEACON <cellIdx> <strength> | MESSAGE <text>
            #print("WAIT")

MyBoard = Board()
MyBoard.loopGame()
