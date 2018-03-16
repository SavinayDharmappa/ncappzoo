MODEL_NAME?=mnist_deep
MODEL_CKPT=$(MODEL_NAME)
OUTPUT_NODE_NAME?=output

YELLOW='\033[1;33m'
NOCOLOR='\033[0m'

.PHONY: all
all: profile 

check_tf:
ifndef TF_SRC_PATH
	$(error TF_SRC_PATH is not defined. Run `export TF_SRC_PATH=path/to/your/tensorflow/source/repo`)
endif

.PHONY: data
data:
	@echo $(YELLOW)"\nDownloading dataset..."$(NOCOLOR)
	@(cd data; wget -nc http://fashion-mnist.s3-website.eu-central-1.amazonaws.com/train-images-idx3-ubyte.gz;)
	@(cd data; wget -nc http://fashion-mnist.s3-website.eu-central-1.amazonaws.com/train-labels-idx1-ubyte.gz;)
	@(cd data; wget -nc http://fashion-mnist.s3-website.eu-central-1.amazonaws.com/t10k-images-idx3-ubyte.gz;)
	@(cd data; wget -nc http://fashion-mnist.s3-website.eu-central-1.amazonaws.com/t10k-labels-idx1-ubyte.gz;)

.PHONY: train
train: data
	@echo $(YELLOW)"\nTraining Fashion MNIST..."$(NOCOLOR)
	(test -f ./model/$(MODEL_CKPT).pb || python3 fashion_mnist_deep.py --data_dir=data --model_dir=model --model_name=$(MODEL_CKPT))

.PHONY: freeze
freeze: check_tf train
	@echo $(YELLOW)"\nFreezing model for inference..."$(NOCOLOR)
	(cd model; test -f $(MODEL_CKPT)_frozen.pb || python3 $(TF_SRC_PATH)/tensorflow/python/tools/freeze_graph.py \
			--input_graph=$(MODEL_CKPT).pb \
			--input_binary=true \
			--input_checkpoint=$(MODEL_CKPT).ckpt \
			--output_graph=$(MODEL_CKPT)_frozen.pb \
			--output_node_name=$(OUTPUT_NODE_NAME);)

.PHONY: compile
compile: freeze
	@echo $(YELLOW)"\nCompiling model to Movidius graph..."$(NOCOLOR)
	(cd model; mvNCCompile -s 12 $(MODEL_CKPT)_frozen.pb -in=input -on=$(OUTPUT_NODE_NAME);)
	@echo $(YELLOW)"Movidius graph generated! You can run inferences using ncappzoo/apps/image-classifier project.\n"$(NOCOLOR)

.PHONY: profile
profile: freeze
	@echo $(YELLOW)"\nProfiling the model..."$(NOCOLOR)
	(cd model; mvNCProfile -s 12 $(MODEL_CKPT)_frozen.pb -in=input -on=$(OUTPUT_NODE_NAME);)
	@echo $(YELLOW)"Movidius graph generated! You can run inferences using ncappzoo/apps/image-classifier project.\n"$(NOCOLOR)

.PHONY: check
check: freeze
	@echo $(YELLOW)"\nComparing results with standard Tensorflow..."$(NOCOLOR)
	(cd model; mvNCCheck -s 12 $(MODEL_CKPT)_frozen.pb -in=input -on=$(OUTPUT_NODE_NAME);)
	@echo $(YELLOW)"Movidius graph generated! You can run inferences using ncappzoo/apps/image-classifier project.\n"$(NOCOLOR)

.PHONY: run
run: compile
	@echo $(YELLOW)"\nRunning inferences using image-classifier project..."$(NOCOLOR)
	(python3 ../../apps/image-classifier/image-classifier.py --graph ./model/graph --dim 28 28 --mean 0 --scale 0.00392 --colormode "monochrome" --labels ./data/categories.txt --image ./data/testing/5/3698.png)

.PHONY: help
help:
	@echo "\nPossible make targets: ";
	@echo "  make help - Shows this message.";
	@echo "  make - Builds all dependencies, but does not run this program.";
	@echo "  make data - Downloads dataset.;"
	@echo "  make train - Start training the neural network.";
	@echo "  make freeze - Export the trained model for inference.";
	@echo "  make compile - Convert the trained model into Movidius graph file.";
	@echo "  make profile - Run the model on NCS and extract complexity, bandwidth and execution time for each layer.";

.PHONY: clean
clean:
	@echo "\nMaking clean...";
	rm -rf model
