# Generative-AI-Applications-with-RAG-and-LangChain-
Capstone Project
Construct the QA bot

In the following sections, you will populate qabot.py with your bot.

Import necessary libraries
Inside qabot.py, import the following from gradio, ibm_watsonx.ai, langchain_ibm, langchain, and langchain_community. The imported classes are necessary for initializing models with the correct credentials, splitting text, initializing a vector store, loading PDFs, generating a question-answer retriever, and using Gradio.

from ibm_watsonx_ai.foundation_models import ModelInference
from ibm_watsonx_ai.metanames import GenTextParamsMetaNames as GenParams
from ibm_watsonx_ai.metanames import EmbedTextParamsMetaNames
from ibm_watsonx_ai import Credentials
from langchain_ibm import WatsonxLLM, WatsonxEmbeddings
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_community.vectorstores import Chroma
from langchain_community.document_loaders import PyPDFLoader
from langchain.chains import RetrievalQA
import gradio as gr
# You can use this section to suppress warnings generated by your code:
def warn(*args, **kwargs):
    pass
import warnings
warnings.warn = warn
warnings.filterwarnings('ignore')

Copied!
Initialize the LLM
You will now initialize the LLM by creating an instance of WatsonxLLM, a class in langchain_ibm. WatsonxLLM can use several underlying foundational models. In this particular example, you will use Mixtral 8x7B, although you could have used other models, such as Llama 3.1 405B. For a list of foundational models available at watsonx.ai, refer to this.

To initialize the LLM, paste the following into qabot.py. Note that you are initializing the model with a temperature of 0.5, and allowing for the generation of a maximum of 256 tokens:

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
## LLM
def get_llm():
    model_id = 'mistralai/mixtral-8x7b-instruct-v01'
    parameters = {
        GenParams.MAX_NEW_TOKENS: 256,
        GenParams.TEMPERATURE: 0.5,
    }
    project_id = "skills-network"
    watsonx_llm = WatsonxLLM(
        model_id=model_id,
        url="https://us-south.ml.cloud.ibm.com",
        project_id=project_id,
        params=parameters,
    )
    return watsonx_llm

Copied!
Define the PDF document loader
Next, you will define the PDF document loader. You will use the PyPDFLoader class from the langchain_community library to load PDF documents. The syntax is quite straightforward. First, you create the PDF loader as an instance of PyPDFLoader. Then, you load the document and return the loaded document. To incorporate the PDF loader in your bot, add the following to qabot.py:

1
2
3
4
5
## Document loader
def document_loader(file):
    loader = PyPDFLoader(file.name)
    loaded_document = loader.load()
    return loaded_document

Copied!
Define the text splitter
The PDF document loader loads the document but does not split it into chunks when using the .load() method. Consequently, you must define a document splitter that will split the text into chunks. Add the following code to qabot.py to define such a text splitter. Note that, in this example, you are defining a RecursiveCharacterTextSplitter with a chunk size of 1000, although other splitters or parameter values are possible:

1
2
3
4
5
6
7
8
9
## Text splitter
def text_splitter(data):
    text_splitter = RecursiveCharacterTextSplitter(
        chunk_size=1000,
        chunk_overlap=50,
        length_function=len,
    )
    chunks = text_splitter.split_documents(data)
    return chunks

Copied!
Define the vector store
Now that you have a way to load the PDF into text and split that text into chunks, you must define a way to embed and store those chunks in a vector database. Add the following code to qabot.py to define a function that embeds the chunks using a yet to be defined embedding model and stores the embeddings in a ChromaDB vector store:

1
2
3
4
5
## Vector db
def vector_database(chunks):
    embedding_model = watsonx_embedding()
    vectordb = Chroma.from_documents(chunks, embedding_model)
    return vectordb

Copied!
Define the embedding model
The above vector_database() function assumes the existence of a watsonx_embedding() function that loads an instance of an embedding model. This embedding model is needed to convert chunks of text into vector representations. The following code defines a watsonx_embedding() function that returns an instance of WatsonxEmbeddings, a class from langchain_ibm that generates embeddings. In this particular case, the embeddings are generated using IBM's Slate 125M English embeddings model. Paste this code into the qabot.py file:

1
2
3
4
5
6
7
8
9
10
11
12
13
## Embedding model
def watsonx_embedding():
    embed_params = {
        EmbedTextParamsMetaNames.TRUNCATE_INPUT_TOKENS: 3,
        EmbedTextParamsMetaNames.RETURN_OPTIONS: {"input_text": True},
    }
    watsonx_embedding = WatsonxEmbeddings(
        model_id="ibm/slate-125m-english-rtrvr",
        url="https://us-south.ml.cloud.ibm.com",
        project_id="skills-network",
        params=embed_params,
    )
    return watsonx_embedding

Copied!
Note that it does not matter to Python that watsonx_embedding() is defined after vector_database(). The order of these definitions could have been reversed, which would have resulted in no change in the underlying functionality of the bot.

Define the retriever
Now that your vector store is defined, you must define a retriever that retrieves chunks of the document from it. In this particular case, you will define a vector store-based retriever that retrieves information using a simple similarity search. To do so, add the following lines to qabot.py:


## Retriever
def retriever(file):
    splits = document_loader(file)
    chunks = text_splitter(splits)
    vectordb = vector_database(chunks)
    retriever = vectordb.as_retriever()
    return retriever


Define a question-answering chain
Finally, it is time to define a question-answering chain! In this particular example, you will use RetrievalQA from langchain, a chain that performs natural-language question-answering over a data source using retrieval-augmented generation (RAG). Add the following code to qabot.py to define a question-answering chain:


## QA Chain
def retriever_qa(file, query):
    llm = get_llm()
    retriever_obj = retriever(file)
    qa = RetrievalQA.from_chain_type(llm=llm, 
                                    chain_type="stuff", 
                                    retriever=retriever_obj, 
                                    return_source_documents=False)
    response = qa.invoke(query)
    return response['result']


Let's recap how all the elements in our bot are linked. Note that RetrievalQA accepts an LLM (get_llm()) and a retriever object (an instance generated by retriever()) as arguments. However, the retriever is based on the vector store (vector_database()), which in turn needed an embeddings model (watsonx_embedding()) and chunks generated using a text splitter (text_splitter()). The text splitter, in turn, needed raw text, and this text was loaded from a PDF using PyPDFLoader. This effectively defines the core functionality of your QA bot!

Set up the Gradio interface
Given that you have created the core functionality of the bot, the final item to define is the Gradio interface. Your Gradio interface should include:

A file upload functionality (provided by the File class in Gradio)
An input textbox where the question can be asked (provided by the Textbox class in Gradio)
An output textbox where the question can be answered (provided by the Textbox class in Gradio)
Add the following code to qabot.py to add the Gradio interface:


# Create Gradio interface
rag_application = gr.Interface(
    fn=retriever_qa,
    allow_flagging="never",
    inputs=[
        gr.File(label="Upload PDF File", file_count="single", file_types=['.pdf'], type="filepath"),  # Drag and drop file upload
        gr.Textbox(label="Input Query", lines=2, placeholder="Type your question here...")
    ],
    outputs=gr.Textbox(label="Output"),
    title="RAG Chatbot",
    description="Upload a PDF document and ask any question. The chatbot will try to answer using the provided document."
)


Add code to launch the application
Finally, you need to add one more line to qabot.py to launch your application using port 7860:

# Launch the app
rag_application.launch(server_name="0.0.0.0", server_port= 7860)

Copied!
After adding the above line, save qabot.py.

Verify qabot.py
Your qabot.py should now look like the following:

from ibm_watsonx_ai.foundation_models import ModelInference
from ibm_watsonx_ai.metanames import GenTextParamsMetaNames as GenParams
from ibm_watsonx_ai.metanames import EmbedTextParamsMetaNames
from ibm_watsonx_ai import Credentials
from langchain_ibm import WatsonxLLM, WatsonxEmbeddings
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_community.vectorstores import Chroma
from langchain_community.document_loaders import PyPDFLoader
from langchain.chains import RetrievalQA
import gradio as gr
# You can use this section to suppress warnings generated by your code:
def warn(*args, **kwargs):
    pass
import warnings
warnings.warn = warn
warnings.filterwarnings('ignore')
## LLM
def get_llm():
    model_id = 'mistralai/mixtral-8x7b-instruct-v01'
    parameters = {
        GenParams.MAX_NEW_TOKENS: 256,
        GenParams.TEMPERATURE: 0.5,
    }
    project_id = "skills-network"
    watsonx_llm = WatsonxLLM(
        model_id=model_id,
        url="https://us-south.ml.cloud.ibm.com",
        project_id=project_id,
        params=parameters,
    )
    return watsonx_llm
## Document loader
def document_loader(file):
    loader = PyPDFLoader(file.name)
    loaded_document = loader.load()
    return loaded_document
## Text splitter
def text_splitter(data):
    text_splitter = RecursiveCharacterTextSplitter(
        chunk_size=1000,
        chunk_overlap=50,
        length_function=len,
    )
    chunks = text_splitter.split_documents(data)
    return chunks
## Vector db
def vector_database(chunks):
    embedding_model = watsonx_embedding()
    vectordb = Chroma.from_documents(chunks, embedding_model)
    return vectordb
## Embedding model
def watsonx_embedding():
    embed_params = {
        EmbedTextParamsMetaNames.TRUNCATE_INPUT_TOKENS: 3,
        EmbedTextParamsMetaNames.RETURN_OPTIONS: {"input_text": True},
    }
    watsonx_embedding = WatsonxEmbeddings(
        model_id="ibm/slate-125m-english-rtrvr",
        url="https://us-south.ml.cloud.ibm.com",
        project_id="skills-network",
        params=embed_params,
    )
    return watsonx_embedding
## Retriever
def retriever(file):
    splits = document_loader(file)
    chunks = text_splitter(splits)
    vectordb = vector_database(chunks)
    retriever = vectordb.as_retriever()
    return retriever
## QA Chain
def retriever_qa(file, query):
    llm = get_llm()
    retriever_obj = retriever(file)
    qa = RetrievalQA.from_chain_type(llm=llm, 
                                    chain_type="stuff", 
                                    retriever=retriever_obj, 
                                    return_source_documents=False)
    response = qa.invoke(query)
    return response['result']
# Create Gradio interface
rag_application = gr.Interface(
    fn=retriever_qa,
    allow_flagging="never",
    inputs=[
        gr.File(label="Upload PDF File", file_count="single", file_types=['.pdf'], type="filepath"),  # Drag and drop file upload
        gr.Textbox(label="Input Query", lines=2, placeholder="Type your question here...")
    ],
    outputs=gr.Textbox(label="Output"),
    title="RAG Chatbot",
    description="Upload a PDF document and ask any question. The chatbot will try to answer using the provided document."
)
# Launch the app
rag_application.launch(server_name="0.0.0.0", server_port= 7860)


