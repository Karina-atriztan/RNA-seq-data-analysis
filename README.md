# RNA-seq-data-analysis
Here I describe the significant steps for RNA-seq data clustering and plots using R studio

###CLUSTERING ANALYSIS USING RNASEA DATA (DIFFERENTIAL EXPRESSED GENES)



#To select input directory
setwd("path/to/input/directory")

outpathcount2 = "path/to/output/directory/"



dir.create(outpathcount2, showWarnings=FALSE)

#We will to use all this dependences to performed the analysis

#BiocManager::install("ggplot2")
#BiocManager::install("ggdendro")
#BiocManager::install("reshape2")
#BiocManager::install("factoextra")
#BiocManager::install("purrr")
#BiocManager::install("dendextend")
#BiocManager::install("dynamicTreeCut")
library("ggplot2")
library("ggdendro")
library("reshape2")
library("grid")
library("factoextra")
library ("multiClust")
library("purrr")
library("dendextend"
library(dynamicTreeCut))
library(gplots)


#Load gene expression matrix
data.exprs <- read.table(file = "all.txt", header = TRUE, sep="\t", comment.char="", row.names = 1)



#This step  is for to scalate the data expresion because genes with high expression can affect the posterior analysis
data.exprs  <- as.matrix(data.exprs)
scaledata <- t(scale(t(data.exprs))) # Centers and scales data.
scaledata <- scaledata[complete.cases(scaledata),]

# Once that data is scalated we performed the clusterization of rows and columns using the herarchical clustering + Pearson and Spearman correlation by cluster
hr <- hclust(as.dist(1-cor(t(scaledata), method="pearson")), method="complete") # Cluster rows by Pearson correlation.
hc <- hclust(as.dist(1-cor(scaledata, method="spearman")), method="complete") # Clusters columns by Spearman correlation.

#Now we’re ready to generate a heatmap of the data. 
#I like heatmap.2 in the gplots package, but you can pass the same call below to the base
#function heatmap() if you like:



heatmap.2(scaledata,
          Rowv=as.dendrogram(hr), 
          #Colv= NA,
          Colv=as.dendrogram(hc),
          col=redblue(100),
          scale="row",
          margins = c(7, 7),
          cexCol = 2,
          key.title = "logFC",
          key.xlab = NA,
          key.ylab = NA,
          keysize = 1,
          density.info = "none",
          labRow = F,
          trace = "none")
	
	
 

#Id you want to extract the column clustering, we performin the nex step
TreeC = as.dendrogram(hc, method="average", )
plot(TreeC,
     main = "Sample Clustering",
     ylab = "Height" 
)

#Also if you are interested in culumn clustering, yo can perform an bootstrap analysis
#This strep is performed when  yuu are interested in to known the similatiry  or differences between columns
boot_hc_cluster <- pvclust(data = scaledata, method.dist = "cor",
                           method.hclust = "average",
                           nboot = 10000, quiet = TRUE)
plot(boot_hc_cluster, cex = 1, print.num = FALSE, cex.pv = 0.6, )

#Now we plot the row dendrogram, using the same approach for column clustering

TreeR = as.dendrogram(hr, method="average")
plot(TreeR,
     leaflab = "none",
     main = "Gene Clustering",
     ylab = "Height",
     horiz = FALSE)

#We can cut the tree high to obtain a small number of large clusters or lower to obtain many small clusters.
#Let’s try a few different cut heights: 1.5, 1.0 and 5.0:
#This step is performed if you  wnat to know thw best number of possible  informative clusers
hclusth1.5 = cutree(hr, h=1.5) #cut tree at height of 1.5
hclusth1.0 = cutree(hr, h=1.0) #cut tree at height of 1.0
hclusth0.5 = cutree(hr, h=0.5) #cut tree at height of 0.5

head (hclusth1.5)
str(hclusth0.5)

library(dendextend)
#plot the tree
plot(TreeR,
     leaflab = "none",
     main = "Gene Clustering",
     ylab = "Height")

#add the three cluster vectors
the_bars <- cbind(hclusth0.5, hclusth1.0, hclusth1.5)
#this makes the bar
colored_bars(the_bars, TreeR, sort_by_labels_order = T, y_shift=-0.1, rowLabels = c("h=0.5","h=1.0","h=1.5"),cex.rowLabels=0.7)
#this will add lines showing the cut heights
abline(h=1.5, lty = 2, col="grey")
abline(h=1.0, lty = 2, col="grey")
abline(h=0.5, lty = 2, col="grey")



##########################Clusterin OPTION 2###################
#using a combinmation of herarchical clusterin and k-means.
# Alternatively, we can designate the number of clusters
#we want by using the ‘k’ option in cutree().
#Here I’m asking for four (k=65): THis, because we before observed that 65 is the best option 
# to analyze the clusters in our sample, it depends of size ans complexity of the sample
hclustk4 = cutree(hr, k=65)
plot(TreeR,
     leaflab = "none",
     main = "Gene Clustering",
     ylab = "Height")
colored_bars(hclustk4, TreeR, sort_by_labels_order = T,
             y_shift=-0.1, rowLabels = c("k=65"),cex.rowLabels=0.7)

####################clustering OPTION 3################################ç
#Other option for obtain clusters, we can use the option cutreeDynamyc from the  "dynamicTreeCut " package. 
#Next method  cut the tree formed by hc using  the distM approach


clusDyn <- cutreeDynamic(hr, distM = as.matrix(as.dist(1-cor(t(scaledata)))),
                         method = "hybrid")

plot(TreeR,
     leaflab = "none",
     main = "Gene Clustering",
     ylab = "Height")

colored_bars(clusDyn, TreeR, sort_by_labels_order = T,
             y_shift=-0.1, rowLabels = NA,
             cex.rowLabels=0.7) #para 5660 genes produce 65 clusters diferentes

#To this moment we opbserved that all clusering methos using produced around 65 clusterrs for out data


####################################################################


# Set the minimum module size
minModuleSize = 50;


###NOW WE GO TO  PERFORMED A CLUSTERIN USING THE WGCNA PACKAGE TO EXTRACT THE FORMED CLUSTERS TO FUTURE ANALYSIS
# Module identification using dynamic tree cut
BiocManager::install("WGCNA")
library(WGCNA)

#WE WILL TO USE THE SAME DENDROGRAM BEFORE CONSTRUCTED AND WILL  CUTTED USING CUTREEDYNAMIC
dynamicMods = cutreeDynamicTree(dendro = hr, maxTreeHeight = 0.5, deepSplit = TRUE, minModuleSize = 25);
#dynamicMods = cutreeDynamic(dendro = geneTree, distM = dissTOM, method="hybrid", deepSplit = 2, pamRespectsDendro = FALSE, minClusterSize = minModuleSize);

#the following command gives the module labels and the size of each module. Lable 0 is reserved for unassigned genes
table(dynamicMods)
dynamicColors = labels2colors(dynamicMods)
table(dynamicColors)
write.table(table(dynamicColors), file="keycolors65K.txt", sep="\t")
plotDendroAndColors(hr, dynamicColors, "k=65", dendroLabels = FALSE,
                    hang = 0.03, addGuide = TRUE, guideHang = 0.05, main = "Gene dendrogram and module colors")


#discard the unassigned genes, and focus on the rest
restGenes= (dynamicColors != "grey")



#Extract modules # extraes el total de modules (cluster) que formaste
#y los guardas con respecto al color que  asisganaste, en nuestro paso, tenemos 65 K, los guarda sin el valor de expresión.
gene.names =row.names(data.exprs)
n=5660
SubGeneNames =gene.names [1:n]
module_colors= setdiff(unique(dynamicColors), "grey")
for (color in module_colors){
  module= SubGeneNames[which(dynamicColors==color)]
  write.table(module, paste("module_",color, ".txt",sep=""), sep="\t", row.names=FALSE, col.names=FALSE,quote=FALSE)
}


#########################################################################################
########################################################################################
#To calculate the cores (aka medoids) of each cluster we can use this function by 
#Biostar user Michael Dundrop. Here I’m using a cutheight of 1.5 to have a smaller 
#number of large clusters.

#Now we can plot the cores as a function of the samples.
#Since this is time series data, we use the time.point.hrs trait to plot the samples.
BiocManager::install("mudata")
library(mudata)
library(ggplot2)
library(reshape2)
#make a dataframe of cores by time

#Let’s perform the actual clsutering using K=4:

set.seed(20)
kClust <- kmeans(scaledata, centers=65, nstart = 1000, iter.max = 20)
kClusters <- kClust$cluster

#Now we can calculate the cluster ‘cores’ aka centroids:

# function to find centroid in cluster i
clust.centroid = function(i, dat, clusters) {
  ind = (clusters == i)
  colMeans(dat[ind,])
}
kClustcentroids <- sapply(levels(factor(kClusters)), clust.centroid, scaledata, kClusters)

#Plotting the centroids to see how they behave:
#library(ggplot2)
#BiocManager::install("reshape")
library(reshape)
#get in long form for plotting
Kmolten <- melt(kClustcentroids)
colnames(Kmolten) <- c('sample','cluster','value')

#plot
p1 <- ggplot(Kmolten, aes(x=sample,y=value, group=cluster, colour=as.factor(cluster))) + 
  geom_point() + 
  geom_line() +
  xlab("Fungivory vs injury") +
  ylab("Expression") +
  labs(title= "Cluster Expression by Time",color = "Cluster")
p1


#So we have some interesting cluster profiles! If you do this analysis and recover
#cores that have very similar expression consider reducing your K.
#An a posteriori means of cluster validation is to correlate the cluster centroids with each other. 
#If the centroids are too similar then they will have a high correlation. 
#If your K number produces clusters with high correlation (say above 0.85) then consider reducing the number of clusters.
cor(kClustcentroids)
#To calculate the scores for a single cluster, in this case 2 we’ll extract the core data for cluster 2,
#then subset the scaled data by cluster =2. Then, we’ll calculate the ‘score’ by correlating each gene
#with the cluster core. We can then plot the results for each gene with the core overlayed:

#Subset the cores molten dataframe so we can plot the core
core2 <- Kmolten[Kmolten$cluster=="49",]

#get cluster 2 #aca´guardar los genes pertenecientes a los clusters ,con su valor de expresion escaldo
K2 <- (scaledata[kClusters==49,])
str(K2)
write.table(K2, file="cluster56.txt", sep="\t")


#calculate the correlation with the core
corscore <- function(x){cor(x,core2$value)}
score <- apply(K2, 1, corscore)
#get the data frame into long format for plotting
K2molten <- melt(K2)#Acá podemos guardar la tabla
colnames(K2molten) <- c('gene','sample','value')
#add the score
K2molten <- merge(K2molten,score, by.x='gene',by.y='row.names', all.x=T)
colnames(K2molten) <- c('gene','sample','value','score')
#order the dataframe by score
#to do this first create an ordering factor
K2molten$order_factor <- 1:length(K2molten$gene)
#order the dataframe by score
K2molten <- K2molten[order(K2molten$gene),]
#set the order by setting the factors
K2molten$order_factor <- factor(K2molten$order_factor , levels = K2molten$order_factor)

# Everything on the same plot
p2 <- ggplot(K2molten, aes(x=sample,y=value)) + 
  geom_line(aes(colour=score, group=gene)) +
  scale_colour_gradientn(colours=c('blue1','red2')) +
  #this adds the core 
  geom_line(data=core2, aes(sample,value, group=cluster), color="black",size =1, inherit.aes=FALSE) +
  xlab("Fungivory/Injury") +
  ylab("Expression") +
  labs(title= "cluster 56",color = "Score", size = 3)
p2 