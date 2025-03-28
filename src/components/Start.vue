<template>
  <div class="app-container">
    <!-- Progress Bar -->
    <div v-if="currentStep === 'survey'" class="progress-bar">
      <div class="progress" :style="{ width: `${progress}%` }"></div>
    </div>

    <div class="content-container">
      <!-- Enqueteur Input Step -->
      <div v-if="currentStep === 'enqueteur'">
        <h2>Prénom enqueteur :</h2>
        <input class="form-control" type="text" v-model="enqueteur" />
        <button
          v-if="enqueteur && !isEnqueteurSaved"
          @click="setEnqueteur"
          class="btn-next"
        >
          Suivant
        </button>
      </div>

      <!-- Start Survey Step -->
      <div v-else-if="currentStep === 'start'" class="start-survey-container">
        <h2>
          Bonjour,<br />
          pour mieux connaître les usagers<br />
          de la gare de Vierzon,<br /><br />
          la Ville et la SNCF souhaiteraient en savoir plus sur votre
          déplacement en cours.<br />
          Auriez-vous quelques secondes à nous accorder ?
        </h2>
        <button @click="startSurvey" class="btn-next">
          COMMENCER QUESTIONNAIRE
        </button>
      </div>

      <!-- Survey Questions Step -->
      <div v-else-if="currentStep === 'survey' && !isSurveyComplete">
        <div class="question-container">
          <h2>{{ currentQuestion.text }}</h2>

          <!-- PDF Button for Q3a and Q3a_nonvoyageur -->
          <button
            v-if="
              ['Q3a', 'Q3b', 'Q3a_nv', 'Q3b_nv', 'Q3a_d', 'Q3b_d'].includes(
                currentQuestion.id
              )
            "
            @click="
              () => {
                console.log('Opening PDF modal');
                console.log('PDF URL:', pdfUrl);
                showPdf = true;
              }
            "
            class="btn-pdf"
          >
            Voir le plan du parking
          </button>

          <!-- Commune Selector for Q2 -->
          <div
            v-if="
              currentQuestion.id === 'Q2' ||
              currentQuestion.id === 'Q2_d' ||
              currentQuestion.id === 'Q2_nv'
            "
          >
            <div
              v-for="(option, index) in currentQuestion.options"
              :key="index"
            >
              <button @click="selectAnswer(option, index)" class="btn-option">
                {{ option.text }}
              </button>
            </div>
          </div>
          <div
            v-else-if="
              currentQuestion.id === 'Q2_precision' ||
              currentQuestion.id === 'Q2d_precision' ||
              currentQuestion.id === 'Q2_precision_nv'
            "
          >
            <CommuneSelector
              v-model="selectedCommune"
              v-model:postalCodePrefix="postalCodePrefix"
            />
            <p>Commune sélectionnée ou saisie: {{ selectedCommune }}</p>
            <button
              @click="handleCommuneSelection"
              class="btn-next"
              :disabled="!selectedCommune.trim()"
            >
              {{ isLastQuestion ? "Terminer" : "Suivant" }}
            </button>
          </div>

          <!-- Street Input -->
          <div v-else-if="currentQuestion.streetInput">
            <div class="input-container">
              <input
                v-model="streetInput"
                class="form-control"
                type="text"
                :placeholder="
                  currentQuestion.freeTextPlaceholder || 'Saisissez une rue'
                "
              />
              <ul v-if="showFilteredStreets" class="commune-dropdown">
                <li
                  v-for="street in filteredStreets"
                  :key="street"
                  @click="selectStreet(street)"
                  class="commune-option"
                >
                  {{ street }}
                </li>
              </ul>
            </div>
            <button
              @click="handleStreetSelection"
              class="btn-next"
              :disabled="!streetInput.trim()"
            >
              {{ isLastQuestion ? "Terminer" : "Suivant" }}
            </button>
          </div>
          <!-- Multiple Choice Questions -->
          <div
            v-else-if="
              !currentQuestion.freeText && !currentQuestion.usesGareSelector
            "
          >
            <div
              v-for="(option, index) in currentQuestion.options"
              :key="index"
            >
              <button @click="selectAnswer(option, index)" class="btn-option">
                {{ option.text }}
              </button>
            </div>
          </div>
          <!-- Gare Selector -->
          <div v-else-if="currentQuestion.usesGareSelector">
            <GareSelector v-model="selectedStation" />
            <button
              @click="handleStationSelection"
              class="btn-next"
              :disabled="!selectedStation.trim()"
            >
              {{ isLastQuestion ? "Terminer" : "Suivant" }}
            </button>
          </div>
          <!-- Free Text Questions -->
          <div v-else>
            <div class="input-container">
              <input
                v-model="freeTextAnswer"
                class="form-control"
                type="text"
                :placeholder="
                  currentQuestion.freeTextPlaceholder || 'Votre réponse'
                "
              />
            </div>
            <button
              @click="handleFreeTextAnswer"
              class="btn-next"
              :disabled="!freeTextAnswer.trim()"
            >
              {{ isLastQuestion ? "Terminer" : "Suivant" }}
            </button>
          </div>
          <!-- Back Button -->
          <button @click="previousQuestion" class="btn-return" v-if="canGoBack">
            Retour
          </button>
          <!-- Partial Validation Button -->
        </div>
      </div>

      <!-- Survey Complete Step -->
      <div v-else-if="isSurveyComplete" class="survey-complete">
        <h2>Merci pour votre réponse et bonne journée.</h2>
        <button @click="resetSurvey" class="btn-next">
          Nouveau questionnaire
        </button>
      </div>

      <!-- Logo -->
      <img class="logo" src="../assets/Alycelogo.webp" alt="Logo Alyce" />
    </div>

    <!-- Footer -->
    <div class="footer">
      <AdminDashboard />
    </div>

    <!-- PDF Modal -->
    <div v-if="showPdf" class="modal">
      <div class="modal-content pdf-content">
        <button
          class="close"
          @click="
            () => {
              showPdf = false;
              console.log('Closing PDF modal');
            }
          "
        >
          ×
        </button>
        <iframe
          :src="pdfUrl"
          width="100%"
          height="500px"
          type="application/pdf"
        >
          This browser does not support PDFs. Please download the PDF to view
          it.
        </iframe>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, watch } from "vue";
import { db } from "../firebaseConfig";
import { collection, getDocs, addDoc } from "firebase/firestore";
import { doc, getDoc, setDoc } from "firebase/firestore";
import { questions } from "./surveyQuestions.js";
import CommuneSelector from "./CommuneSelector.vue";
import AdminDashboard from "./AdminDashboard.vue";
import GareSelector from "./GareSelector.vue";

// Refs
const docCount = ref(0);
const currentStep = ref("enqueteur");
const startDate = ref("");
const enqueteur = ref("");
const currentQuestionIndex = ref(0);
const answers = ref({});
const freeTextAnswer = ref("");
const questionPath = ref(["Q1"]);
const isEnqueteurSaved = ref(false);
const isSurveyComplete = ref(false);
const selectedStation = ref("");
const selectedCommune = ref("");
const postalCodePrefix = ref("");
const showPdf = ref(false);
const pdfUrl = ref("/Plan.pdf");
const stationInput = ref("");
const streetInput = ref("");
const filteredStations = ref([]);
const filteredStreets = ref([]);
const selectedPoste = ref(null);

// Firestore refs
const surveyCollectionRef = collection(db, "Vierzon");
const counterDocRef = doc(db, "counters", "surveyCounter");

// Stations list

const stationsList = [
  "Amboise",
  "Ancenis",
  "La Angers-Saint-Laud-Landerneau",
  "Batz-sur-Mer",
  "Baule",
  "Blois-Chambord",
  "Chaingy-Fourneaux-Plage",
  "Chouzy",
  "La Baule-Escoublac",
  "La Baule-les-Pins",
  "La Chapelle-Saint-Mesmin",
  "La Chaussée-Saint-Victor",
  "Le Croisic",
  "Le Pouliguen",
  "Les Aubrais",
  "Limeray",
  "Menars",
  "Mer",
  "Meung-sur-Loire",
  "Montlouis-sur-Loire",
  "Nantes",
  "Noizay",
  "Onzain-Chaumont-sur-Loire",
  "Orléans",
  "Pornichet",
  "Saint-Ay",
  "Saint-Nazaire",
  "Saint-Pierre-des-Corps",
  "Saumur",
  "Suèvres",
  "Tours",
  "Veuves-Monteaux",
];

const streetsList = [
  "Allée Auguste Rodin",
  "Allée Camille Blanc",
  "Allée Camille Claudel",
  "Allée Georges Charpak",
  "Allée Germaine Tillion",
  "Allée Pierre de Coubertin",
  "Allée Pierre-gilles de Gennes",
  "Allee Andre Rethore",
  "Allee de la Source",
  "Allee des Chataigniers",
  "Allee des Longueraies",
  "Allee Guillaume Apollinaire",
  "Allee Hoche",
  "Avenue de Chaillot",
  "Avenue de la République",
  "Avenue de Verdun",
  "Avenue du 14 Juillet",
  "Avenue du 19 Mars 1962",
  "Avenue du 8 Mai 1945",
  "Avenue du Colonel Manhes",
  "Avenue du Jonc",
  "Avenue Edouard Vaillant",
  "Avenue Henri Brisson",
  "Avenue Jean Jaurès",
  "Avenue Mal Philippe Leclerc de Ha",
  "avenue maréchal jean de lattre de tassigny",
  "avenue maréchal philippe leclerc de hauteclocque",
  "Avenue Pierre Billes",
  "Avenue Pierre Sémard",
  "Boulevard de la Liberté",
  "Boulevard de la Nation",
  "Boulevard Salvador Allende",
  "Chemin Blanc",
  "Chemin de Bois Marteau",
  "Chemin de Charnay",
  "Chemin de Fougery",
  "Chemin de Grands Champs",
  "Chemin de la Bazinière",
  "Chemin de la Bercetterie",
  "Chemin de la Bidauderie",
  "Chemin de la Bruyère",
  "Chemin de la Coudre",
  "Chemin de la Cour aux Taureaux",
  "Chemin de la Craillo",
  "Chemin de la Croix",
  "Chemin de la Fosse au Gue",
  "Chemin de la Gaillardière",
  "Chemin de la Giraudiere",
  "Chemin de la Gratouille",
  "Chemin de la Jonchere",
  "Chemin de la Jouannetterie",
  "Chemin de la Maison Blanche",
  "Chemin de la Petite Noue",
  "Chemin de la Picardière",
  "Chemin de la Prairie de Langlee",
  "Chemin de la Touche",
  "Chemin de la Trietterie",
  "Chemin de l'Abricot",
  "Chemin de l'Ardillat",
  "Chemin de l'Etang du Verdin",
  "Chemin de l'Orme a Lieue",
  "Chemin de Saint Priest",
  "Chemin des Ajoncs",
  "Chemin des Assis",
  "Chemin des Bergeries",
  "Chemin des Bertes",
  "Chemin des Chavannes",
  "Chemin des Communaux de la Vevre",
  "Chemin des Crêles",
  "Chemin des Erables",
  "Chemin des Fondereaux",
  "Chemin des Fougeres",
  "Chemin des Fromenteaux",
  "Chemin des Gaudrets",
  "Chemin des Gdes Noues Fontaines",
  "Chemin des Genievres",
  "Chemin des Grandchamps",
  "Chemin des Grandes Crêles",
  "Chemin des Grandes Terres",
  "Chemin des Grands Bois",
  "Chemin des Joffrois",
  "Chemin des Millards",
  "Chemin des Noues",
  "Chemin des Paincourts",
  "Chemin des Peages",
  "Chemin des Pelles",
  "Chemin des Petites Crêles",
  "Chemin des Petites Vèves",
  "Chemin des Pillots",
  "Chemin des Rechignardes",
  "Chemin des Riaux",
  "Chemin des Sables de Saint Prié",
  "Chemin des Tailles",
  "Chemin des Terres du Verdin",
  "Chemin des Vallees",
  "Chemin des Varennes",
  "Chemin du Briou",
  "Chemin du Camp",
  "Chemin du Carroir aux Ajoncs",
  "Chemin du Champ Chapelet",
  "Chemin du Champ des Renards",
  "Chemin du Chevry",
  "Chemin du Cloître",
  "Chemin du Crot du Lac",
  "Chemin du Gardois",
  "Chemin du Grand Orme",
  "Chemin du Jouffroy",
  "Chemin du Perdrier",
  "Chemin du Pre d'Anet",
  "Chemin du Pre Long",
  "Chemin du Stade Constant Duval",
  "Chemin du Terrichon",
  "Chemin du Tertre",
  "Chemin du Verdin",
  "Chemin du Village d'En Haut",
  "Chemin Pierre Dupont",
  "Chemin Ruisseau des Rechignardes",
  "Cheminement de Grand Champs",
  "Clos de la Minee",
  "Cour des Donneaux",
  "Cours de Fougery",
  "Grande Rue de la Genette",
  "Hameau de la Dumonerie",
  "Ile de Labras",
  "Imp Ancienne Cite de la Verrerie",
  "Imp des Castors de la Jonchere",
  "Impasse Andre de Chenier",
  "Impasse Andre Ribaud",
  "Impasse Armand Bazile",
  "Impasse Babeuf",
  "Impasse Barbes",
  "Impasse Baudin",
  "Impasse Casimir Lecomte",
  "Impasse de Bel Air",
  "Impasse de Bellevue",
  "Impasse de Fay",
  "Impasse de Jérusalem",
  "Impasse de la Craillo",
  "Impasse de la Croix Moreau",
  "Impasse de la Gaucherie",
  "Impasse de la Glaciere",
  "Impasse de la Nation",
  "Impasse de Vieilfond",
  "Impasse des Feuillures",
  "Impasse des Fontaines",
  "Impasse des Fossés",
  "Impasse des Grands Moulins",
  "Impasse des Lunots",
  "Impasse des Tailles",
  "Impasse Diderot",
  "Impasse du Clos Chabot",
  "Impasse du Clos du Roy",
  "Impasse du Grelet",
  "Impasse Etienne Marcel",
  "Impasse Felix Pyat",
  "Impasse George Sand",
  "Impasse Jean Rostand",
  "impasse jean-baptiste clément",
  "Impasse Karl Marx",
  "Impasse Kléber",
  "Impasse Mozart",
  "Impasse Pierre Debournou",
  "Impasse Raspail",
  "Impasse Riparia",
  "Impasse Saint Leonard",
  "Impasse Suzanne Masson",
  "La Cour d'En Haut",
  "La Giraudière",
  "La Grande Loeuf",
  "La Rondonniere",
  "L'Aujonniere",
  "le Bâtonnet",
  "le Petit Râteau",
  "Le Village aux Richoux",
  "Les Grandes Joncheres",
  "Les Varennes",
  "Levee de l'Etang du Verdin",
  "Lotissement de la Colline",
  "Parc de Bellevue",
  "Passage de l'Abricot",
  "Passage de l'Aubepine",
  "Passage de l'Hospice",
  "Passage de Ronvaux",
  "Passage des Bergeries",
  "Passage des Bergeries Est",
  "Passage des Bergeries Ouest",
  "Passage des Epinettes",
  "Passage Descartes",
  "Passage du Bourdoiseau",
  "Passage du Chambon",
  "Passage du Chateau",
  "Passage du Crot à Foulon",
  "Passage du Jonc",
  "Passage du Pré d'Anet",
  "Passage Flourens",
  "Passage Jean Jacques Rousseau",
  "Passage Leveque",
  "Passage Parmentier",
  "Passage Proudhon",
  "Passage Tissier",
  "Passage Voltaire",
  "Petit Chemin de Bois Marteau",
  "Petit Chemin de la Giraudiere",
  "Petit Chemin Petit Chem de Bois Marteau",
  "Petite Rue Anatole France",
  "Petite Rue Babeuf",
  "Petite Rue de Fay",
  "Petite Rue des Donneaux",
  "Petite Rue des Longueraies",
  "Petite Rue du Bourdoiseau",
  "Petite Rue du Camp",
  "Petite Rue du Chambon",
  "Place Aimé Cesaire",
  "Place Ambroise Croizat",
  "Place Aristide Briand",
  "Place de la Genette",
  "Place de la Libération",
  "Place de la Mémoire",
  "Place de la République",
  "Place de la Résistance",
  "Place de l'Hotel de Ville",
  "Place de l'Eglise des Forges",
  "Place de l'Etape",
  "Place du Bas de Grange",
  "Place du Château",
  "Place du Maréchal Foch",
  "Place du Tacot",
  "Place François Mitterrand",
  "Place Gabriel Péri",
  "Place Gallerand",
  "Place Jacques Brel",
  "Place Marceau",
  "Place Roland Champenier",
  "Place Salvador Allende",
  "Place Salvatore Allende",
  "Place Vaillant Couturier",
  "Quai de l'Etang",
  "Quai du Bassin",
  "Quai du Cher",
  "Quai d'Yèvre",
  "Résidence Anatole France",
  "Résidence de la Poste",
  "Résidence des Coumelins",
  "Résidence le Clos de la Minée",
  "Res Mermoz Nord",
  "Residence d'Hery",
  "Residence de la Foret",
  "Residence des Bruyeres",
  "Residence les Dumaines",
  "Residence les Moulins Verts",
  "Residence les Pommiers",
  "Residence Mermoz",
  "Residence Mermoz Sud",
  "Residence Val d'Yevre",
  "Rive du Cher Nord Est",
  "Route d'Ainset",
  "Route de Bellon",
  "Route de Bonègue",
  "Route de Bourges",
  "Route de Foëcy",
  "Route de Guérigny",
  "Route de la Croix",
  "Route de la Loeuf",
  "Route de Méreau",
  "Route de Massay",
  "Route de Neuvy",
  "Route de Puits Berteau",
  "Route de Saint Laurent",
  "Route de Saint Lazare",
  "Route de Tours",
  "Route des Noues",
  "Route des Souchènes",
  "Route du Bois Blanc",
  "Route du Petit Râteau",
  "Route du Vieux Domaine",
  "Route du Village d'En Haut",
  "Route René Dumont",
  "Rue Abbé Pierre",
  "Rue Adolphe Hache",
  "Rue Alain Fournier",
  "Rue Albert et Paul Thouvenin",
  "Rue Albert Schweitzer",
  "Rue Alfred Nobel",
  "Rue Alphonse Daudet",
  "Rue Alphonse Pradat",
  "Rue Ambroise Croizat",
  "Rue Anatole France",
  "Rue André Gide",
  "Rue André Guillemain",
  "Rue André Hènault",
  "Rue André Ribaud",
  "Rue Andre de Chenier",
  "Rue Anne Franck",
  "Rue Antoine Parmentier",
  "Rue Antonin Lerault",
  "Rue Aristide Briand",
  "Rue Armand Bazile",
  "Rue Armand Brunet",
  "Rue Arthur Rimbaud",
  "Rue Auber",
  "Rue aux Chats",
  "Rue Babeuf",
  "Rue Baptiste Marcet",
  "Rue Bara",
  "Rue Barbès",
  "Rue Baudelaire",
  "Rue Beau Site",
  "Rue Beethoven",
  "Rue Bernard Dumont",
  "Rue Bernard Palissy",
  "Rue Berty Albrecht",
  "Rue Blaise Pascal",
  "Rue Blanche Baron",
  "Rue Blanqui",
  "Rue Bobby Sands",
  "Rue Boitard",
  "Rue Célestin Gérard",
  "Rue Camelinat",
  "Rue Camille Desmoulins",
  "Rue Casimir Lecomte",
  "Rue Charles Gibault",
  "Rue Charles Hurvoy",
  "Rue Charlotte Delbo",
  "Rue Claude Bernard",
  "Rue Claude Chevalier",
  "Rue Claude Levi-strauss",
  "Rue Claude Michaud",
  "Rue Cognacq Jay",
  "Rue Condorcet",
  "Rue Courbet",
  "Rue Danielle Casanova",
  "Rue Danton",
  "Rue de Bellevue",
  "Rue de Bitterfeld",
  "Rue de Fay",
  "Rue de Jérusalem",
  "Rue de la Bidauderie",
  "Rue de la Convention",
  "Rue de la Croix Moreau",
  "Rue de la Fringale",
  "Rue de la Gaucherie",
  "Rue de la Monnaie",
  "Rue de la Montagne",
  "Rue de la Paix",
  "Rue de la Petite Noue",
  "Rue de la Petite Vitesse",
  "Rue de la Piscine",
  "Rue de la Plaisance",
  "Rue de la Poterie",
  "Rue de la Société Française",
  "Rue de la Voûte",
  "Rue de l'Abricot",
  "Rue de l'Armistice",
  "Rue de l'Epargne",
  "Rue de l'Etape",
  "Rue de l'Hospice",
  "Rue de l'Ile du Cher",
  "Rue de l'Unité",
  "Rue de Rendsburg",
  "Rue de Ronvaux",
  "Rue de Stalingrad",
  "Rue de Valmy",
  "Rue de Villeneuve",
  "Rue de Wittelsheim",
  "Rue Denis Papin",
  "Rue des Ateliers",
  "Rue des Berlurettes",
  "Rue des Changes",
  "Rue des Donneaux",
  "Rue des Ecoles",
  "Rue des Epinettes",
  "Rue des Etablissements Merlin",
  "Rue des Feuillures",
  "Rue des Francs Tireurs",
  "Rue des Hortensias",
  "Rue des Lilas",
  "Rue des Longueraies",
  "Rue des Meunières",
  "Rue des Ormes",
  "Rue des Pentecotes",
  "Rue des Pillots",
  "Rue des Ponts",
  "Rue des Seringats",
  "Rue des Terres Mortes",
  "Rue des Tramways de l'Indre",
  "Rue Descartes",
  "Rue d'Héry",
  "Rue Diderot",
  "Rue du 11 Novembre 1918",
  "Rue du Bas de Grange",
  "Rue du Bois d'Yèvre",
  "Rue du Bourdoiseau",
  "Rue du Bourgneuf",
  "Rue du Cavalier",
  "Rue du Château",
  "Rue du Château d'Eau",
  "Rue du Champ Commun",
  "Rue du Champ du Four",
  "Rue du Champanet",
  "Rue du Chemin Vert",
  "Rue du Colonel Fabien",
  "Rue du Crot à Foulon",
  "Rue du Docteur Lobligeois",
  "Rue du Docteur Pierre Roux",
  "Rue du Général de Gaulle",
  "Rue du Grand Clement",
  "Rue du Grelet",
  "Rue du Gros Caillou",
  "rue du maréchal foch",
  "Rue du Maréchal Joffre",
  "Rue du Mouton",
  "Rue du Péry",
  "Rue du Panorama",
  "Rue du Plessis",
  "Rue du Presbytère",
  "Rue du President Wilson",
  "Rue du Putet",
  "Rue du Sergent Bobillot",
  "Rue du Souvenir Français",
  "Rue du Square Emile Peraudin",
  "Rue du Travail",
  "Rue du Val Fleury",
  "Rue du Verdin",
  "Rue du Vert Pommier",
  "Rue du Village Aubry",
  "Rue Edgar Quinet",
  "Rue Edmond Rostand",
  "Rue Edouard Branly",
  "Rue Eirick Labonne",
  "Rue Elisee Reclus",
  "Rue Elsa Triolet",
  "Rue Emile Accolas",
  "Rue Emile Bouleau",
  "Rue Emile Desforges",
  "Rue Emile Zola",
  "Rue Ernest Renan",
  "Rue Etienne Desroches",
  "Rue Etienne Dolet",
  "Rue Etienne Marcel",
  "Rue Etienne Nivet",
  "Rue Eugène Pottier",
  "Rue Evariste Galois",
  "Rue Félix Pyat",
  "Rue Federico Garcia Lorca",
  "Rue Fernand Léger",
  "Rue Fourier",
  "Rue Frédéric Mistral",
  "Rue François Mitterrand",
  "Rue France Bloch-sérazin",
  "Rue Gérard Philipe",
  "Rue Gagarine",
  "Rue Galilée",
  "Rue Gallerand",
  "Rue Gambetta",
  "Rue Gambon",
  "Rue Garibaldi",
  "Rue Gaston Cornavin",
  "Rue Gaston Monmousseau",
  "Rue Gay Lussac",
  "Rue George Sand",
  "Rue Georges Clémenceau",
  "Rue Georges Fauconnier",
  "Rue Georges Politzer",
  "Rue Georges Rousseau",
  "Rue Gisèle Halimi",
  "Rue Gourdon",
  "Rue Grelon",
  "Rue Gustave Flaubert",
  "Rue Gustave Flourens",
  "Rue Guy MÃ´quet",
  "Rue Hector Berlioz",
  "Rue Henri Barbusse",
  "Rue Henri Bergson",
  "Rue Henri Martin",
  "Rue Henri Sellier",
  "Rue Honoré de Balzac",
  "Rue Jacqueline Auriol",
  "Rue Jacques Brel",
  "Rue Jacques Decour",
  "Rue Jacques Richard",
  "Rue Jean Cocteau",
  "Rue Jean de la Fontaine",
  "Rue Jean Giono",
  "Rue Jean Jacques Rousseau",
  "Rue Jean Moulin",
  "Rue Jean Picot",
  "Rue Jean Racine",
  "Rue Jean Vilar",
  "Rue Jean-baptiste Clément",
  "Rue Jeanne Labourbe",
  "Rue Joliot Curie",
  "Rue Jules Guesde",
  "Rue Jules Louis Breton",
  "Rue Jules Vallès",
  "Rue Jules Verne",
  "Rue Julian Grimau",
  "Rue Lénine",
  "Rue Léo Mérigot",
  "Rue Léon Bourgeois",
  "Rue Lavoisier",
  "Rue Ledru Rollin",
  "Rue Louise Michel",
  "Rue Mac Nab",
  "Rue Marat",
  "Rue Marcel Cachin",
  "Rue Marcel Pagnol",
  "Rue Marcel Paul",
  "Rue Marcel Perrin",
  "Rue Marcel Sembat",
  "Rue Maryse Bastié",
  "Rue Mattéoti",
  "Rue Meunier",
  "Rue Michel Lucas",
  "Rue Michelet",
  "Rue Millière",
  "Rue Mirabeau",
  "Rue Miranda de Ebro",
  "Rue Molière",
  "Rue Monge",
  "Rue Mozart",
  "Rue Nicolas Boileau",
  "Rue Nicolas Copernic",
  "Rue Pablo Casals",
  "Rue Pablo Neruda",
  "Rue Pablo Picasso",
  "Rue Pasteur",
  "Rue Paul Doumer",
  "Rue Paul Eluard",
  "Rue Paul Lafargue",
  "Rue Paul Valery",
  "Rue Paul Verlaine",
  "Rue Pierre Brossolette",
  "Rue Pierre Calmette",
  "Rue Pierre Curie",
  "Rue Pierre Debournou",
  "Rue Pierre Dupont",
  "Rue Pierre et Jean Serpaud",
  "Rue Pierre Ferdonnet",
  "Rue Porte aux Boeufs",
  "Rue Proudhon",
  "Rue Rabelais",
  "Rue Raspail",
  "Rue Riparia",
  "Rue Robert Barran",
  "Rue Robert Desnos",
  "Rue Robespierre",
  "Rue Roger Faletto",
  "Rue Roland Champenier",
  "Rue Rouget de Lisle",
  "Rue Saint Exupéry",
  "Rue Théodore Roosevelt",
  "Rue Theodore Monod",
  "Rue Urbain Nègre",
  "Rue Van Craeynest",
  "Rue Victor Hugo",
  "Rue Vincent Auriol",
  "Rue Voltaire",
  "Rue Xavier de Maistre",
  "Ruelle du Chevrier",
  "Ruelle du Corneau",
  "Ruelle du Pavillon",
  "Square Georges Brassens",
  "Square Henriette Dumuin",
  "Square Jacques Duclos",
  "square jean-baptiste clément",
  "Square Louis Braille",
  "Village au Piquet",
  "Voie Communale Z I de l'Aujonniere",
  "ZA Pole d'Echanges A71",
];

// Computed properties
const currentQuestion = computed(() => {
  return currentQuestionIndex.value >= 0 &&
    currentQuestionIndex.value < questions.length
    ? questions[currentQuestionIndex.value]
    : null;
});

const showFilteredStations = computed(
  () => stationInput.value.length > 0 && filteredStations.value.length > 0
);

const showFilteredStreets = computed(
  () => streetInput.value.length > 0 && filteredStreets.value.length > 0
);

const canGoBack = computed(() => questionPath.value.length > 1);

const isLastQuestion = computed(
  () => currentQuestionIndex.value === questions.length - 1
);

const progress = computed(() => {
  if (currentStep.value !== "survey") return 0;
  if (isSurveyComplete.value) return 100;
  const totalQuestions = questions.length;
  const currentQuestionNumber = currentQuestionIndex.value + 1;
  const isLastOrEnding =
    isLastQuestion.value ||
    currentQuestion.value?.options?.some((option) => option.next === "end");
  return isLastOrEnding
    ? 100
    : Math.min(Math.round((currentQuestionNumber / totalQuestions) * 100), 99);
});

const isValidCommuneSelection = computed(() => {
  return (
    selectedCommune.value.includes(" - ") || selectedCommune.value.trim() !== ""
  );
});

// Add these new methods
const filterStations = () => {
  const input = stationInput.value.toLowerCase();
  filteredStations.value = stationsList.filter((station) =>
    station.toLowerCase().includes(input)
  );
};

const filterStreets = () => {
  const input = streetInput.value.toLowerCase();
  filteredStreets.value = streetsList.filter((street) =>
    street.toLowerCase().includes(input)
  );
};

const selectStation = (station) => {
  stationInput.value = station;
  filteredStations.value = [];
};

const selectStreet = (street) => {
  streetInput.value = street;
  filteredStreets.value = [];
};

// Methods
const setEnqueteur = () => {
  if (enqueteur.value.trim() !== "") {
    currentStep.value = "start";
    isEnqueteurSaved.value = true;
  }
};

const startSurvey = () => {
  startDate.value = new Date().toLocaleTimeString("fr-FR", {
    hour: "2-digit",
    minute: "2-digit",
    second: "2-digit",
  });
  currentStep.value = "survey";
  answers.value = {};
  isSurveyComplete.value = false;
  questionPath.value = ["Q1"]; // Start with Q1
  currentQuestionIndex.value = 0;
};

const selectAnswer = (option, index) => {
  answers.value[currentQuestion.value.id] = index + 1;

  // Special handling for Q2 and Q2_nonvoyageur
  if (
    currentQuestion.value.id === "Q2" ||
    currentQuestion.value.id === "Q2_nv" ||
    currentQuestion.value.id === "Q2_d"
  ) {
    if (index === 0) {
      // "Vierzon" selected
      const questionPrefix =
        currentQuestion.value.id === "Q2"
          ? "Q2"
          : currentQuestion.value.id === "Q2_d"
          ? "Q2_d"
          : "Q2_nv";
      answers.value[`Q2_COMMUNE`] = "VIERZON";
      answers.value["CODE_INSEE"] = "18279";
      answers.value["COMMUNE_LIBRE"] = "";
    }
  }

  if (option.next === "end") {
    finishSurvey();
  } else if (option.requiresPrecision) {
    nextQuestion(option.next);
  } else {
    nextQuestion();
  }
};

const handleFreeTextAnswer = () => {
  if (currentQuestion.value) {
    // Skip for street questions since they're handled by handleStreetSelection
    if (
      currentQuestion.value.id === "Q2a" ||
      currentQuestion.value.id === "Q2a_d" ||
      currentQuestion.value.id === "Q2a_nv"
    ) {
      return;
    }

    answers.value[currentQuestion.value.id] = freeTextAnswer.value;
    if (currentQuestionIndex.value < questions.length - 1) {
      nextQuestion();
    } else {
      finishSurvey();
    }
  }
};

const handleStationSelection = () => {
  if (selectedStation.value.trim() !== "") {
    answers.value[currentQuestion.value.id] = selectedStation.value;
    nextQuestion();
    selectedStation.value = ""; // Reset for next use
  }
};

const handleStreetSelection = () => {
  if (streetInput.value.trim() !== "") {
    const isListedStreet = streetsList.includes(streetInput.value);
    const questionId = currentQuestion.value.id;

    // Store the answer based on the question ID
    if (questionId === "Q2a") {
      answers.value["Q2a"] = streetInput.value;
      if (!isListedStreet) {
        answers.value["Q2a_CUSTOM"] = streetInput.value;
      }
    } else if (questionId === "Q2a_d") {
      answers.value["Q2a_d"] = streetInput.value;
      if (!isListedStreet) {
        answers.value["Q2a_d_CUSTOM"] = streetInput.value;
      }
    } else if (questionId === "Q2a_nv") {
      answers.value["Q2a_nv"] = streetInput.value;
      if (!isListedStreet) {
        answers.value["Q2a_nv_CUSTOM"] = streetInput.value;
      }
    }

    // Force move to next question
    const nextQuestionId = currentQuestion.value.next;
    if (nextQuestionId === "end") {
      finishSurvey();
    } else {
      const nextIndex = questions.findIndex((q) => q.id === nextQuestionId);
      if (nextIndex !== -1) {
        currentQuestionIndex.value = nextIndex;
        questionPath.value.push(nextQuestionId);
      }
    }

    // Reset the input
    streetInput.value = "";
    filteredStreets.value = [];
  }
};

// Add these watches
watch(stationInput, () => {
  filterStations();
});

watch(streetInput, () => {
  filterStreets();
});

const updateSelectedCommune = (value) => {
  selectedCommune.value = value;
};

const handleCommuneSelection = () => {
  if (selectedCommune.value.trim() !== "") {
    const parts = selectedCommune.value.split(" - ");
    const currentQuestionId = currentQuestion.value.id;
    const isNonPassenger = currentQuestionId.includes("nonvoyageur");
    const questionPrefix = isNonPassenger ? "Q2_nonvoyageur" : "Q2";

    if (parts.length === 2) {
      // Dropdown selection
      const [commune, codeInsee] = parts;
      answers.value[`${questionPrefix}_COMMUNE`] = commune;
      answers.value["CODE_INSEE"] = codeInsee;
      answers.value["COMMUNE_LIBRE"] = ""; // Clear COMMUNE_LIBRE
    } else {
      // Manual entry or free text
      answers.value[`${questionPrefix}_COMMUNE`] = ""; // Clear the dropdown commune
      answers.value["CODE_INSEE"] = ""; // Clear INSEE code
      answers.value["COMMUNE_LIBRE"] = selectedCommune.value.trim(); // Set COMMUNE_LIBRE
    }
    nextQuestion();
  }
};

const nextQuestion = (forcedNextId = null) => {
  let nextQuestionId = forcedNextId;
  if (!nextQuestionId && currentQuestion.value) {
    if (
      currentQuestion.value.usesGareSelector ||
      currentQuestion.value.freeText
    ) {
      nextQuestionId = currentQuestion.value.next;
    } else {
      const selectedAnswer = answers.value[currentQuestion.value.id];
      if (currentQuestion.value.id === "Poste") {
        nextQuestionId = "Q1";
      } else {
        const selectedOption =
          currentQuestion.value.options[selectedAnswer - 1];
        nextQuestionId = selectedOption?.next || null;
      }
    }
  }

  if (nextQuestionId === "end") {
    finishSurvey();
  } else if (nextQuestionId) {
    const nextIndex = questions.findIndex((q) => q.id === nextQuestionId);
    if (nextIndex !== -1) {
      currentQuestionIndex.value = nextIndex;
      questionPath.value.push(nextQuestionId);
      freeTextAnswer.value = "";
      selectedCommune.value = "";
      postalCodePrefix.value = "";
    }
  }
};

const previousQuestion = () => {
  if (canGoBack.value) {
    // Remove current question from path
    const currentQuestionId = questionPath.value.pop();
    const previousQuestionId =
      questionPath.value[questionPath.value.length - 1];

    // Find indices
    const previousIndex = questions.findIndex(
      (q) => q.id === previousQuestionId
    );

    if (previousIndex !== -1) {
      // Update current question index
      currentQuestionIndex.value = previousIndex;

      // Clear current question's answers
      if (currentQuestionId) {
        // Clear main answer
        delete answers.value[currentQuestionId];

        // Clear any custom/additional fields
        delete answers.value[`${currentQuestionId}_CUSTOM`];

        // Clear special fields for commune questions
        if (currentQuestionId.includes("Q2")) {
          delete answers.value["Q2_COMMUNE"];
          delete answers.value["CODE_INSEE"];
          delete answers.value["COMMUNE_LIBRE"];
        }
      }

      // Reset all input fields
      freeTextAnswer.value = "";
      stationInput.value = "";
      streetInput.value = "";
      selectedCommune.value = "";
      postalCodePrefix.value = "";

      // Clear filtered lists
      filteredStations.value = [];
      filteredStreets.value = [];
    }
  }
};

const finishSurvey = async () => {
  isSurveyComplete.value = true;
  const now = new Date();
  const uniqueId = await getNextId();

  // Determine if it's a passenger or non-passenger survey
  const isPassenger = answers.value["Q1"] === 1;
  const questionPrefix = isPassenger ? "Q2" : "Q2_nonvoyageur";

  // Create answers object
  const orderedAnswers = {
    ID_questionnaire: uniqueId,
    HEURE_DEBUT: startDate.value,
    DATE: now.toLocaleDateString("fr-FR").replace(/\//g, "-"),
    JOUR: now.toLocaleDateString("fr-FR", { weekday: "long" }),
    ENQUETEUR: enqueteur.value,
    HEURE_FIN: now.toLocaleTimeString("fr-FR", {
      hour: "2-digit",
      minute: "2-digit",
      second: "2-digit",
    }),
    TYPE_QUESTIONNAIRE: isPassenger ? "Passager" : "Non-passager",
    [`${questionPrefix}_COMMUNE`]:
      answers.value[`${questionPrefix}_COMMUNE`] || "",
    CODE_INSEE: answers.value["CODE_INSEE"] || "",
  };

  // Add all answers
  Object.keys(answers.value).forEach((key) => {
    orderedAnswers[key] = answers.value[key];
  });

  await addDoc(surveyCollectionRef, orderedAnswers);
  await getDocCount();
};

const resetSurvey = () => {
  currentStep.value = "start";
  startDate.value = "";
  answers.value = {};
  currentQuestionIndex.value = 0;
  questionPath.value = ["Q1"];
  freeTextAnswer.value = "";
  isSurveyComplete.value = false;
  // Note: We don't clear selectedPoste or sessionStorage
};

const getDocCount = async () => {
  try {
    const querySnapshot = await getDocs(surveyCollectionRef);
    docCount.value = querySnapshot.size;
  } catch (error) {
    console.error("Error getting document count:", error);
  }
};

const getNextId = async () => {
  const counterDoc = await getDoc(counterDocRef);
  let counter = 1;

  if (counterDoc.exists()) {
    counter = counterDoc.data().value + 1;
  }

  await setDoc(counterDocRef, { value: counter });

  return `Vierzon-${counter.toString().padStart(6, "0")}`;
};

// Add computed property for showing partial validation button
const showPartialValidation = computed(() => {
  if (!currentQuestion.value) return false;

  // Check if we're on the correct path based on Q1 answer
  const isPassengerPath = answers.value["Q1"] === 1;
  const isDescendedPath = answers.value["Q1"] === 2;

  // Get the relevant Q5 question ID based on the path
  const relevantQ5 = isPassengerPath ? "Q5" : isDescendedPath ? "Q5_d" : null;

  if (!relevantQ5) return false;

  // Find the index of the relevant Q5
  const q5Index = questions.findIndex((q) => q.id === relevantQ5);
  if (q5Index === -1) return false;

  // Show button if we're at or past the relevant Q5 and have valid input
  return (
    currentQuestionIndex.value >= q5Index &&
    (currentQuestion.value.id === relevantQ5
      ? stationInput.value.trim() !== ""
      : currentQuestion.value.freeText
      ? freeTextAnswer.value.trim() !== ""
      : currentQuestion.value.streetInput
      ? streetInput.value.trim() !== ""
      : true)
  );
});

// Modify handlePartialValidation to save current answer before finishing
const handlePartialValidation = async () => {
  // Save current answer based on question type
  if (currentQuestion.value) {
    if (
      currentQuestion.value.id === "Q5" ||
      currentQuestion.value.id === "Q5_d"
    ) {
      const isListedStation = stationsList.includes(stationInput.value);
      if (currentQuestion.value.id === "Q5") {
        answers.value["Q5"] = stationInput.value;
        if (!isListedStation) {
          answers.value["Q5_CUSTOM"] = stationInput.value;
        }
      } else {
        answers.value["Q5_d"] = stationInput.value;
        if (!isListedStation) {
          answers.value["Q5_d_CUSTOM"] = stationInput.value;
        }
      }
    } else if (currentQuestion.value.streetInput) {
      const isListedStreet = streetsList.includes(streetInput.value);
      const questionId = currentQuestion.value.id;
      answers.value[questionId] = streetInput.value;
      if (!isListedStreet) {
        answers.value[`${questionId}_CUSTOM`] = streetInput.value;
      }
    } else if (currentQuestion.value.freeText) {
      answers.value[currentQuestion.value.id] = freeTextAnswer.value;
    }
  }

  // Save the survey
  await finishSurvey();

  // Start a new survey immediately
  startSurvey();
};

// Lifecycle hooks
onMounted(() => {
  getDocCount();
});
</script>


<style>
/* Base styles */
body {
  background-color: #2a3b63;
  margin: 0;
  padding: 0;
  font-family: Arial, sans-serif;
}

.app-container {
  display: flex;
  flex-direction: column;
  min-height: 100vh;
  background-color: #2a3b63;
  color: white;
}

.content-container {
  flex-grow: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 5% 0;
  width: 90%;
  max-width: 600px;
  margin: 0 auto;
  box-sizing: border-box;
  overflow-y: auto;
}

.question-container {
  width: 100%;
  margin-bottom: 30px;
}

.input-container,
.station-input-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  width: 100%;
  position: relative;
}

.form-control {
  width: 100%;
  max-width: 400px;
  padding: 10px;
  border-radius: 5px;
  border: 1px solid white;
  background-color: #333;
  color: white;
  font-size: 16px;
  margin-bottom: 15px;
}

.btn-next,
.btn-return,
.btn-option,
.btn-pdf {
  width: 100%;
  max-width: 400px;
  color: white;
  padding: 15px;
  margin-top: 10px;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  font-size: 16px;
}

.btn-next {
  background-color: green;
}

.btn-return {
  background-color: grey;
  margin-top: 30px;
}

.btn-option {
  background-color: #4a5a83;
  text-align: left;
}

.btn-pdf {
  background-color: #3498db;
  margin: 15px auto;
  display: block;
}

.commune-dropdown {
  position: relative;
  width: 100%;
  max-width: 400px;
  max-height: 200px;
  overflow-y: auto;
  background-color: #333;
  border: 1px solid #666;
  border-radius: 5px;
  z-index: 1000;
  margin: -10px auto 15px;
  padding: 0;
  list-style: none;
}

.commune-option {
  padding: 10px;
  cursor: pointer;
  color: white;
  border-bottom: 1px solid #444;
}

.commune-option:hover {
  background-color: #444;
}

.station-input-container {
  position: relative;
  width: 100%;
  max-width: 400px;
  margin: 0 auto;
}

.logo {
  max-width: 25%;
  height: auto;
  margin-top: 40px;
  margin-bottom: 20px;
}

.footer {
  background: linear-gradient(to right, #4c4faf, #3f51b5);
  padding: 20px;
  text-align: center;
  width: 100%;
  box-sizing: border-box;
}

.progress-bar {
  width: 100%;
  height: 10px;
  background-color: #e0e0e0;
  position: relative;
  overflow: hidden;
  margin-bottom: 20px;
}

.progress {
  height: 100%;
  background-color: #4caf50;
  transition: width 0.3s ease-in-out;
}

@media screen and (max-width: 480px) {
  .commune-dropdown {
    width: 90%;
    left: 50%;
    transform: translateX(-50%);
  }

  .form-control {
    max-width: 90%;
  }
}

.modal {
  position: fixed;
  z-index: 9999;
  left: 0;
  top: 0;
  width: 100%;
  height: 100%;
  background-color: #1a1a1a;
  display: flex;
  justify-content: center;
  align-items: center;
}

.modal-content {
  width: 100%;
  height: 100%;
  position: relative;
  background-color: #1a1a1a;
}

.pdf-content {
  width: 100%;
  height: 100%;
  position: relative;
}

.pdf-content iframe {
  border: none;
  width: 100%;
  height: 100%;
  background: white;
}

.close {
  position: fixed;
  right: 20px;
  top: 20px;
  width: 40px;
  height: 40px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: transparent;
  border: none;
  cursor: pointer;
  z-index: 10000;
  opacity: 0.7;
  transition: opacity 0.2s;
}

.close::before,
.close::after {
  content: "";
  position: absolute;
  width: 24px;
  height: 2px;
  background-color: white;
  transform-origin: center;
}

.close::before {
  transform: rotate(45deg);
}

.close::after {
  transform: rotate(-45deg);
}

.close:hover {
  opacity: 1;
}

@media screen and (min-width: 768px) {
  .modal {
    padding: 40px;
  }

  .modal-content {
    max-width: 1200px;
    margin: 0 auto;
  }
}

.btn-partial {
  width: 100%;
  max-width: 400px;
  color: white;
  padding: 15px;
  margin-top: 10px;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  font-size: 16px;
  background-color: #ff9800;
}
</style>
