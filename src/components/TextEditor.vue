<template>
  <v-container class="editor-container">
    <v-card class="editor-card">
      <v-card-title class="text-center text-xl font-bold">Anotações da sessão</v-card-title>
      <v-card-text class="d-flex flex-column align-center justify-start p-0 overflow-auto">
        <v-text-field v-model="patientName" label="Nome do Paciente" clearable outlined class="w-full mb-4"
          :rules="[v => !!v || 'O nome do paciente é obrigatório']"></v-text-field>

        <div ref="editor" class="editor-content"></div>

        <v-btn :disabled="!isFormValid" color="primary" @click="onSave" class="w-full mt-4">
          Salvar
        </v-btn>

        <v-btn color="secondary" @click="dialog = true" class="w-full mt-2">
          Ver Anotações
        </v-btn>
      </v-card-text>
    </v-card>

    <v-dialog v-model="dialog" max-width="800px">
      <v-card>
        <v-card-title>
          <span class="text-lg font-semibold">Registros de Anotações</span>
          <v-spacer></v-spacer>
          <v-btn icon @click="dialog = false">
            <v-icon>mdi-close</v-icon>
          </v-btn>
        </v-card-title>

        <v-card-text>
          <v-row>
            <v-col cols="12" sm="6">
              <v-text-field v-model="search" label="Buscar por nome do paciente" clearable></v-text-field>
            </v-col>
            <v-col cols="12" sm="6">
              <v-text-field v-model="dateFilter" label="Filtrar por data" type="date" clearable></v-text-field>
            </v-col>
          </v-row>

          <v-data-table :headers="headers" :items="filteredRecords" :items-per-page="5" class="elevation-1"
            no-data-text="Nenhum registro encontrado">
            <template v-slot:item.notes="{ item }">
              <div v-html="item.notes"></div> <!-- ✅ Exibe HTML com formatação básica -->
            </template>

            <template v-slot:item.actions="{ item }">
              <v-btn icon @click="deleteRecord(item.id)" color="red" class="btn-delete">
                <v-icon>mdi-delete</v-icon>
              </v-btn>
            </template>
          </v-data-table>
        </v-card-text>

        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn color="primary" @click="dialog = false">Fechar</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
  </v-container>
</template>

<script>
import Quill from 'quill';
import 'quill/dist/quill.snow.css';
import dayjs from 'dayjs';
import utc from 'dayjs/plugin/utc';
import timezone from 'dayjs/plugin/timezone';
import { jsPDF } from 'jspdf';
import html2canvas from 'html2canvas';

dayjs.extend(utc);
dayjs.extend(timezone);

export default {
  name: 'TextEditor',
  data() {
    return {
      content: '',
      patientName: '',
      editor: null,
      dialog: false,
      search: '',
      dateFilter: '',
      records: [
        { id: 1, patient: 'Ana Souza', date: '2024-02-20', notes: 'Paciente apresentou melhora significativa.' },
        { id: 2, patient: 'Carlos Lima', date: '2024-02-18', notes: 'Sessão focada em estratégias de enfrentamento.' },
        { id: 3, patient: 'Beatriz Mendes', date: '2024-02-15', notes: 'Relatou dificuldades no ambiente de trabalho.' },
      ],
      headers: [
        { text: 'Nome do Paciente', value: 'patient' },
        { text: 'Data', value: 'date' },
        { text: 'Anotações', value: 'notes' },
        { text: 'Ações', value: 'actions', sortable: false },
      ],
    };
  },

  computed: {
    isFormValid() {
      return this.patientName.trim().length > 0 && this.content.trim().length > 0;
    },
    filteredRecords() {
      return this.records.filter((record) => {
        const matchesName = record.patient
          .toLowerCase()
          .includes(this.search.toLowerCase());

        const recordDate = dayjs(record.date).local().format('YYYY-MM-DD');
        const filterDate = this.dateFilter
          ? dayjs(this.dateFilter).local().format('YYYY-MM-DD')
          : null;

        console.log('Comparando:', { recordDate, filterDate });

        const matchesDate = filterDate ? recordDate === filterDate : true;
        return matchesName && matchesDate;
      });
    }
  },

  mounted() {
    this.editor = new Quill(this.$refs.editor, {
      theme: 'snow',
      placeholder: 'Digite aqui...',
      modules: {
        toolbar: [
          [{ 'header': '1' }, { 'header': '2' }, { 'font': [] }],
          [{ 'list': 'ordered' }, { 'list': 'bullet' }],
          ['bold', 'italic', 'underline'],
          ['link', 'image'],
          [{ 'align': [] }],
        ],
      },
    });

    this.editor.on('text-change', () => {
      this.content = this.editor.root.innerHTML;
    });

    this.editor.root.addEventListener('click', (event) => {
      if (event.target && event.target.tagName === 'IMG') {
        event.target.style.maxWidth = '100%';
        event.target.style.height = 'auto';
      }
    });
  },

  methods: {
    cleanQuillHTML(html) {
      return html
        .replace(/<span[^>]*>(.*?)<\/span>/g, '$1')      // Remove spans desnecessários
        .replace(/ data-list="[^"]*"/g, '')             // Remove atributos data-list
        .replace(/<p><br><\/p>/g, '')                   // Remove parágrafos vazios
        .replace(/\s{2,}/g, ' ')                       // Remove espaços extras
        .trim();                                       // Mantém tags essenciais (<ol>, <li>, <strong>, etc.)
    },
    formatDate(date) {
      return dayjs(date).local().format('DD/MM/YYYY');
    },
    onSave() {
      if (!this.content.trim() || !this.patientName.trim()) {
        console.error("Erro: Nome do paciente e anotações são obrigatórios.");
        return;
      }

      const formattedContent = this.content;
      const localDate = dayjs().utc().format('YYYY-MM-DD');

      // 📝 Criar um elemento temporário para renderizar o HTML
      const tempDiv = document.createElement("div");
      tempDiv.innerHTML = `<h3>Paciente: ${this.patientName}</h3>${formattedContent}`;
      tempDiv.style.width = '100%';
      document.body.appendChild(tempDiv);

      // 🖨️ Gerar PDF preservando formatação
      html2canvas(tempDiv, { scale: 2 }).then((canvas) => {
        const imgData = canvas.toDataURL('image/png');
        const pdf = new jsPDF('p', 'mm', 'a4');
        const imgProps = pdf.getImageProperties(imgData);
        const pdfWidth = pdf.internal.pageSize.getWidth();
        const pdfHeight = (imgProps.height * pdfWidth) / imgProps.width;
        pdf.addImage(imgData, 'PNG', 0, 0, pdfWidth, pdfHeight);
        pdf.save(`Anotacao-${this.patientName}.pdf`);

        document.body.removeChild(tempDiv);
      });

      // 💾 Salvar o registro mantendo as formatações básicas
      const cleanContent = this.cleanQuillHTML(this.content);

      const newRecord = {
        id: this.records.length + 1,
        patient: this.patientName,
        date: localDate,
        notes: cleanContent,
      };
      this.records.push(newRecord);

      console.log('Registro salvo:', newRecord);

      this.patientName = '';
      this.editor.root.innerHTML = '';
      this.content = '';
    },

    deleteRecord(id) {
      this.records = this.records.filter((record) => record.id !== id);
      console.log(`Registro com ID ${id} excluído.`);
    },
  },
};
</script>

<style scoped>
.editor-container {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100vh;
  background: transparent !important;
  padding: 0;
  margin: 0;
}

.editor-card {
  height: 600px;
  width: 700px;
  background-color: white !important;
  border-radius: 16px;
  box-shadow: 0px 4px 10px rgba(0, 0, 0, 0.2);
  display: flex;
  flex-direction: column;
  font-family: 'Poppins', sans-serif;
}

.editor-content {
  min-height: 300px;
  width: 100%;
  overflow-y: auto;
  font-family: 'Poppins', sans-serif;
  font-weight: 500;
  font-size: 16px;
  color: #333333;
}

.editor-content img {
  max-width: 100%;
  height: auto;
  object-fit: contain;
  display: block;
  margin: 10px auto;
}

.ql-editor.ql-blank::before {
  color: #a0a0a0;
  font-style: italic;
  font-family: 'Poppins', sans-serif;
  font-weight: 400;
}

.btn-delete {
  width: 24px !important;
  height: 24px !important;
  min-width: 24px !important;
  padding: 0 !important;
}

.btn-delete .v-icon {
  font-size: 16px !important;
}
</style>
