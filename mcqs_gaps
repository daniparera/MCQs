import textwrap, argparse

from nltk.corpus import wordnet as wn
import os, sys, time
from lxml import etree
from collections import OrderedDict

from random import randint
import random

import wikipedia

from nltk.corpus import wordnet_ic
brown_ic = wordnet_ic.ic('ic-brown.dat')
semcor_ic = wordnet_ic.ic('ic-semcor.dat')

#import pattern.en
from nltk.stem import WordNetLemmatizer

def isplural(word):
	lemma = WordNetLemmatizer().lemmatize(word, 'n')
	isp = True if word is not lemma else False
	return isp

def plural(word):
	if word.endswith('y'):
		return word[:-1] + 'ies'
	elif word[-1] in 'sx' or word[-2:] in ['sh', 'ch']:
		return word + 'es'
	elif word.endswith('an'):
		return word[:-2] + 'en'
	else:
		return word + 's'

#input=raw_input

parserarg = argparse.ArgumentParser(
	 prog='MCQs',
	 formatter_class=argparse.RawDescriptionHelpFormatter,
	 description=textwrap.dedent('''\
		 generate questions from a text/naf file
		 --------------------------------

			 example of use $python3 %(prog)s [[--naf]] --file input.txt [[--debug]]
		 '''))

parserarg.add_argument('--debug', action='store_false', default='TRUE', help='to show aditional information (default is false)')
parserarg.add_argument('--naf', action='store_false', default='TRUE', help='to generate naf file (default is false)')

parserarg.add_argument('--distractors', dest='distractors', required=False, default=3, type=int , help='# distractors (default 3)')
parserarg.add_argument('--file', dest='input_file', required=False, default='', type=str , help='file to be analyses')
parserarg.add_argument('--topic', dest='topic', required=False, default='Solar_System', type=str , help='wikipedia topic')
parserarg.add_argument('--depth', dest='depth', required=False, default=8, type=int , help='how many sentences in wikipedia?')

args = parserarg.parse_args()

debug = not(bool(args.debug))
naf = not(bool(args.naf))

number_distractors = args.distractors
fname = args.input_file
topic = args.topic
depth = args.depth

if (len(sys.argv) < 2 or len(sys.argv) > 7):
	print ("Incorrect Parameters")
	print ("Usage: python " + sys.argv[0] + " [[--file input_file | --wiki input_topic --depth value]]")
	print ("Usage: python " + sys.argv[0] + " [[--naf]] [[--coref]] --file input_file [[--debug]]")
	print ("Help: python " + sys.argv[0] + " --help")
	exit(1)

if(depth and topic):
	try:
		page = wikipedia.summary(topic, sentences=depth)

	except wikipedia.exceptions.PageError as e:
		print topic+" may refer to:"

	except wikipedia.exceptions.DisambiguationError:
	#except wikipedia.exceptions.DisambiguationError as e:
		#for elem in e.options: print elem
		#exit(-1)

		topics = wikipedia.search(topic)
		print topic+" may refer to: "
		for i, topic in enumerate(topics):
			print i, topic
		choice = int(raw_input("Enter a choice: "))
		assert choice in xrange(len(topics))
		page = wikipedia.summary(topics[choice])
	except: # catch *all* exceptions
		e = sys.exc_info()[0]
		print( "<p>Error: %s</p>" % e )
		exit(-1)

	tmp_file = open("wiki.txt", "w")
	if debug: sys.stderr.write(str(page)+"\n")
	tmp_file.write(page)
	tmp_file.close()
	fname = "wiki.txt"

elif(not fname):
	print ("Need input_file or input topic")
	print ("Usage: python " + sys.argv[0] + " [[--file input_file | --wiki input_topic --depth value]]")
	print ("Usage: python " + sys.argv[0] + " [[--naf]] [[--coref]] --file input_file [[--debug]]")
	print ("Help: python " + sys.argv[0] + " --help")
	exit(-1)

# generate (or not) NAF file from input #####

if naf:

	naf_name = fname.split('.')[0]+".naf"

	cmd = "cat "+fname   
	cmd = cmd + " | java -jar resources/ixa-pipe-tok-1.7.0.jar tok -l en "
	cmd = cmd + " | java -jar resources/ixa-pipe-pos-1.3.3.jar tag -m resources/en-maxent-100-c5-baseline-dict-penn.bin "
	cmd = cmd + " > "+naf_name

	if debug: sys.stderr.write(str(cmd)+"\n")
	os.system(cmd)

	fname = naf_name

naf_file = open(fname, "r")
chk_file = open(fname, "r")

##############################################

# check that input file is NAF file ##########

chk = chk_file.readline().strip()

if not chk == "<NAF xml:lang=\"en\" version=\"2.0\">" and not chk == "<?xml version=\"1.0\" encoding=\"UTF-8\"?>":
	print("Incorrect NAF format file")
	exit(-1)

chk_file.close()

##############################################

# Support class for questions ################

class Question(object):

	def __repr__(self):
		return "\nS: "+str(self.stem)+"\nK: "+str(self.key)+"\nDS: "+", ".join([str(i) for i in self.distractors])+"\n"

	def __str__(self):
		return "\nS: "+str(self.stem)+"\nK: "+str(self.key)+"\nDS: "+", ".join([str(i) for i in self.distractors])+"\n"

	def __init__(self, key, stem):
		self.key = key
		self.stem = stem
		self.distractors = []

	def insertDistractor(self, distractor):
		self.distractors.append(distractor)

class Answer(object):

	def __init__(self, num, correct, res):
		self.num = num
		self.correct = correct
		self.res = res

##############################################

question = []

questions = []

adjList = {}
verbList = {}
subjList = {}

stems = []
stem = []
sent = 1	#initial 

parser = etree.XMLParser(remove_blank_text=True) # discard whitespace nodes
tree = etree.parse(naf_file, parser)
root = tree.getroot() # document element

###########################################################################################################################
################################################# FILL the GAP # ##########################################################
######################### Get all text, split in sentence, select keys and generate distractors ###########################
###########################################################################################################################

# STEM selection. for all the words in NAF file separate every sentence by sentence number ################################
for word in root.xpath("text/wf"):

	if debug: sys.stderr.write(str(sent))
	if debug: sys.stderr.write(word.get("sent"))
	if sent != int(word.get("sent")):
		stems.append(stem)
		stem = []
		sent = sent + 1

	stem.append(word.text)

for st in stems:
	if debug: sys.stderr.write("STEMS: ")
	if debug: sys.stderr.write(str(st))
###########################################################################################################################

# CANDIDATES selection. ###################################################################################################

# Get candidatesLists, all the nouns in a sentence, later one of them would be select to be a key.
candidatesLists = []
candidates = {}

# sentence number, stored in NAF file. sentence number is used to split text in sentences.
sent = 1

# for all the terms in NAF file separate by sentence and select only the nouns.
for term in root.xpath("terms/term"):

	if debug: sys.stderr.write(str(sent)+"\n")
	if debug: sys.stderr.write(str(term.xpath("span/target"))+"\n")
	t_id = term.xpath("span/target")[0].get("id")
	word = root.xpath("text/wf[@id='"+t_id+"']")[0]

	if debug: sys.stderr.write(str(word.get("sent"))+"\n")
	if sent != int(word.get("sent")):
		candidatesLists.append(candidates)
		candidates = {}
		sent = sent + 1

	# use lemma as a key. Lemma is used for Wornet processing, to calculate similarity
	if term.get("pos") == "N" or term.get("pos") == "R":
		candidates[term.get("lemma")] = [0.0,word.text] ### lemma is the key, value is a list -> [number,word] 

for cd in candidatesLists:
	if debug: sys.stderr.write("CANDIDATES: ")
	if debug: sys.stderr.write(str(cd)+"\n")

###########################################################################################################################

if debug: sys.stderr.write("----------------------------------------\n")

# KEY selection. ##########################################################################################################

keys = []

# For all the lemmas selected, compare one element each other, build an ordered list. select randomly one with highest value
for candidates in candidatesLists:
	if debug: sys.stderr.write(str(candidates)+"\n")

	for candidate1 in candidates:

		if debug: sys.stderr.write(str(candidate1)+"\n")
		subjectSynsetsList = wn.synsets(candidate1, pos='n')

		if(len(subjectSynsetsList)>0): # if the subject list of synset isn't empty

			subjectSynsetElem = subjectSynsetsList[0]

			for candidate2 in candidates:

				nounSynsetElem = wn.synsets(candidate2, pos='n')

				if(len(nounSynsetElem)>0): # if the subject list of synset isn't empty

					lin = subjectSynsetElem.lin_similarity(nounSynsetElem[0], semcor_ic)  
					path = subjectSynsetElem.path_similarity(nounSynsetElem[0])  
					wup = subjectSynsetElem.wup_similarity(nounSynsetElem[0])  

					# update similarity value
					candidates[candidate1][0] = candidates[candidate1][0] + wup + path + lin

	if debug: sys.stderr.write(str(candidates)+"\n")
	candidatesSorted = OrderedDict(sorted(candidates.items(), key=lambda t: t[1], reverse=True))
	if debug: sys.stderr.write("NEXT------------------------------------------------")
	rand = int(randint(0, len(candidatesSorted)/4))
	#rand = 0
	if debug: sys.stderr.write(str(candidatesSorted.items()[rand])+"\n")
	if debug: sys.stderr.write(str(candidatesSorted.items()[rand][0])+"\n")
	if debug: sys.stderr.write(str(candidatesSorted.items()[rand][1][1])+"\n")
	keys.append([candidatesSorted.items()[rand][0],candidatesSorted.items()[rand][1][1]])
		#candidate = []

###########################################################################################################################

# Replace keys with blank in STEMS ########################################################################################
stems_join = ""

for idx,k in enumerate(keys):
	if debug: sys.stderr.write("CANDIDATES: \n")
	if debug: sys.stderr.write("[lemma,word]: "+str(k)+"\n")
	if debug: sys.stderr.write("stem: "+str(stems[idx])+"\n")

	stems_join = " ".join(stems[idx]).replace(k[1], '________',1)

	questions.append(Question(k[1],stems_join))
	stems_join = ""

if debug: sys.stderr.write("QUESTIONS: \n")
if debug: sys.stderr.write(str(questions)+"\n")

###########################################################################################################################

# Distractors generation ##################################################################################################

distractorsNounList = {}

# get all the nouns from the text, this would be the distractors
for term in root.xpath("terms"):

	for subelem in term.xpath("term"):

		if subelem.get("pos") == "N" or subelem.get("pos") == "R": 
			distractorsNounList[subelem.get("lemma")] = 0.0

if debug: sys.stderr.write(str(distractorsNounList)+"\n")

# For each selected key, compare it with all the distractor list generate (step before), select three with highest value
for idx,values in enumerate(keys):

	key_lemma = values[0]
	key_word = values[1]
 
	if debug: sys.stderr.write(str(key_lemma.istitle()))
	if debug: sys.stderr.write("COMPARE BASE --> "+key_lemma+"\n")

	subjectSynsetsList = wn.synsets(key_lemma, pos='n')

	if(len(subjectSynsetsList)>0): # if the subject list of synset isn't empty

		subjectSynsetElem = subjectSynsetsList[0]

		distractorsTemp = {}
		distractorsTemp2 = {}
		distractorsTemp3 = {}

		for distractor in distractorsNounList:

			# Concordance between key and distractor, both (or neither) must be proper noun
			if key_lemma.istitle() == distractor.istitle():

				if debug: sys.stderr.write(str(distractor.istitle()))
				if debug: sys.stderr.write("WITH --> "+distractor+"\n")

				distractorElem = wn.synsets(distractor, pos='n')

				if(len(distractorElem)>0): # if the subject list of synset isn't empty

					lin = subjectSynsetElem.lin_similarity(distractorElem[0], semcor_ic)  
					path = subjectSynsetElem.path_similarity(distractorElem[0])  
					wup = subjectSynsetElem.wup_similarity(distractorElem[0])  

					distractorsTemp[distractor] = wup
					distractorsTemp2[distractor] = path
					distractorsTemp3[distractor] = lin

					#if debug: sys.stderr.write("NEXT ITM"

	distractorsTempSorted = OrderedDict(sorted(distractorsTemp.items(), key=lambda t: t[1], reverse=True))
	distractorsTempSorted2 = OrderedDict(sorted(distractorsTemp2.items(), key=lambda t: t[1], reverse=True))
	distractorsTempSorted3 = OrderedDict(sorted(distractorsTemp3.items(), key=lambda t: t[1], reverse=True))
	#sorted(hyperList, )

	if debug: sys.stderr.write("NEXT------------------------------------------------\n")
	rand = int(randint(0, len(distractorsTempSorted)/4))
	#rand = 0
	if debug: sys.stderr.write(str(distractorsTempSorted)+"\n")

	number_distractor = 0

	# Add distractors to the questions structured data, check corncordance between keys and distractors
	for t in distractorsTempSorted:
		if debug: sys.stderr.write("PROCESING DISTRACTOR --> "+str(t)+", TO INSERT IN QUESTIONS LIST\n")

		# distractor not must be in stem and in key list 
		if not t in stems[idx] and not t in keys[idx]:

			# if key is capitalized distractor must be capitalized too
			if key_word.istitle() : t = t.capitalize() 

			# if key is plural distractor must be plural too
			if isplural(key_word) :  t = plural(t)

			questions[idx].insertDistractor(t)
			number_distractor = number_distractor + 1
		
		# how many distractors we need??
		if number_distractor == number_distractors:
			break

#exit()

if debug: sys.stderr.write("UPDATED WITH DISTRACTORS QUESTIONS: \n")
if debug: sys.stderr.write(str(questions)+"\n")

if debug: sys.stderr.write("############################################################# \n\n")

def getKey(item):
	return item[0]

############################ QUESTIONNAIRE PART ###########################################################################

score = 0

previous_time = time.time()
elapsed_time = 0

for elem in questions:

	# construct index for answer arrar ################################################################################
	distractors = len(elem.distractors)																				###
	#distractors = number_distractors	, deprecated, it's more flexible other option. distractor = -1				###
																													###
	myrandoms = random.sample(xrange(1, distractors+2), distractors+1)												###
	#myrandoms = random.sample(xrange(1, number_distractors+2), number_distractors+1)								###
	######################################################### #########################################################

	if debug: sys.stderr.write(str(myrandoms)+"\n")

	# construct array of answers, correct/key at last position ########################################################
	ans = []																										###
																													###
	for idx,val in enumerate(myrandoms[:-1]):																		###
		ans.append((val,elem.distractors[idx]))																		###
																													###
	ans.append((myrandoms[-1],elem.key))																			###
	######################################################### #########################################################

	# order answers to don't give a clue ##############################################################################
	ans_ord =  sorted(ans, key=getKey)																				###
	######################################################### #########################################################

	if debug: sys.stderr.write(str(ans)+"\n")
	if debug: sys.stderr.write(str(ans_ord)+"\n")

	######################################################### #########################################################
	######################################################### #########################################################

	# construct and make the question to user #########################################################################
	res = ""																										###
	for rs in ans_ord:																								###
		res = res + str(rs[0]) + ")" + rs[1] + "  "																	###
																													###
	choice = raw_input("QUESTION:\n"+elem.stem+"\n"+res+"\n"+"YOUR ANSWER IS: ")									###
																													###
	# answer must be a single integer																				###
	if choice.strip() == "": choice = -1																			###
	elif len(choice)!= 1 and len(choice)!= 2: choice = -2															###
	elif not choice.isdigit(): choice = -2																			###
																													###
	######################################################### #########################################################

	# while answer is not correct, repeat it ##########################################################################
	while not int(choice) in myrandoms:																				###
		if choice == -1:																							###
			print("\n\nPlease. Select some option!! \n")															###
		else:																										###
			print("\n\nPlease. Select one correct option!! \n")														###
																													###
		choice = raw_input("QUESTION:\n"+elem.stem+"\n"+res+"\n"+"YOUR ANSWER IS:")									###
																													###
		# answer must be a single integer																			###
		if choice.strip() == "": choice = -1																		###
		elif len(choice)!= 1 and len(choice)!= 2: choice = -2														###
		elif not choice.isdigit(): choice = -2																		###
																													###
	######################################################### #########################################################

	# when user gets a valid answer, compare if answer is correct or not ##############################################
	if int(choice) == ans[-1][0]:																					### 
		result = "Correct"																							###
		score = score + 1																							###
	else :																											###
		result = "Incorrect, correct is number "+str(ans[-1][0])													###
																													###
	######################################################### #########################################################

	# finish, presentation of correct answer and calculation of elapsed time ##########################################
	print("So, you answer is...")																					###
																													###
	# time elapsed calculation																						###
	elapsed_time = elapsed_time + time.time() - previous_time														###
																													###
	time.sleep(1)																									###
	print("\n"+result + "!\n")																						###
																													###
	if debug: sys.stderr.write("TIME: "+str(elapsed_time)+"\n")														###
	raw_input("Press Enter to continue...")																			###
																													###
	print(" ")																										###
	previous_time = time.time()																						###
	######################################################### #########################################################

minutes, seconds = divmod(elapsed_time, 60)

if minutes < 10: minutes = "0"+str(int(minutes))
else: minutes = str(int(minutes))

if seconds < 10: seconds = "0"+str(int(seconds))
else: seconds = str(int(seconds))

print_txt = "You grade is {}! You spend {}\":{}' to complet {} questions."
print(print_txt).format(10*score/len(questions),minutes,seconds,len(questions))
