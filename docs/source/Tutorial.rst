Tutorial
========

Every year, the Pandora Team runs part of the UK LArTPC Software and Analysis Workshop. There will be a full introduction and tutorial slides on there. The 2026 workshop can be found `here <https://indico.ph.ed.ac.uk/event/434/sessions/684/#20261027>`_.

This page will cover the basics which would allow you to make your own Pandora edits/algorithms for your own analysis as needed. 

Generally, Pandora is run as part of the reco2 reconstruction stage, or consolidated reco ( reco1 + reco2). Thus, you will need an input at either the reco1 or detsim stage ready. Reco2 is the higher level reconstruction, which does things like particle creation. 

Basic Config Edits
------------------

Each stage is controlled via fcl files. They can :code:`#include` other fcl files to steer different parts of the reconstruction. You can see everything that is set by a particular fcl by using fhicl-dump:

.. code-block::

    $ fhicl-dump standard_reco2_sbnd.fcl | less -p "pandora:"

So this command shows you how Pandora is being steered from the standard reco2 fcl of SBND. 

You can then make your own fcl file to change the Pandora steering to be whatever you want. To do this, :code:`#include` the standard reco2 fcl file to sort out everything else. Then you can set the commands of :code:`physics.producers.pandora` to be true or false, for example:

.. code-block::

    #include "standard_reco2_sbnd.fcl"

    physics.producers.pandora.ShouldRunCosmicRecoOption: false
    physics.producers.pandora.ShouldRunNeutrinoRecoOption: true

Running this then allows you to make basic configuration edits to Pandora within reco2:

.. code-block::

   lar -c my_fcl_file.fcl -n -1 -s my_reco1_file.root


XML Config Edits
----------------

By running the fhicl-dump above, you should have seen that Pandora is controlled via an xml config file. This can also be modified. You can add a line in your custom fcl to change the xml being pointed to:

.. code-block::
   physics.producers.pandora.ConfigFile: "PandoraSettings_Master_SBND_MyVersion.xml"

Normally the Pandora xml settings live in various places, which you can git clone down to make your own copy. A copy of most of them can be found in `LArReco <https://github.com/PandoraPFAOrg/LArReco/tree/master>`_ / settings.

In these you can edit which algorithms run and in what order. For example, if you wanted to see exactly which algorithms were running you could set :code:`ShouldDisplayAlgorithmInfo` in the Global Settings to true.

.. code-block::

   <!-- GLOBAL SETTINGS -->
    <IsMonitoringEnabled>true</IsMonitoringEnabled>
    <ShouldDisplayAlgorithmInfo>false</ShouldDisplayAlgorithmInfo>
    <SingleHitTypeClusteringMode>true</SingleHitTypeClusteringMode>

New Algorithms
--------------

You want to put these in `LArContent <https://github.com/PandoraPFAOrg/LArContent>`_.

In :code:`larpandoracontent/larpandoracontent`, you must choose where to put your algorithm. Beware: if you choose to make your own folder, make sure it is added to the :code:`LArContent.cc` file. 

Say you choose to add your algorithm within :code:`LArMonitoring`. You can see the structure of many algorithms within this folder, but the key thing is that they all consist of a .cc file and a .h file.

You can use any of these pairs as an example. Here we will use the :code:`VertexMonitoringAlgorithm`. The key parts of the .cc file of a new algorithm are:

- The set up. This will be a number of :code:`#include` files, and introducing the namespaces.

.. code-block::

    #include "Pandora/AlgorithmHeaders.h"

    #include "larpandoracontent/LArMonitoring/VertexMonitoringAlgorithm.h"

    #include "larpandoracontent/LArHelpers/LArGeometryHelper.h"
    #include "larpandoracontent/LArHelpers/LArInteractionTypeHelper.h"
    #include "larpandoracontent/LArHelpers/LArMCParticleHelper.h"
    #include "larpandoracontent/LArHelpers/LArPfoHelper.h"
    #include "larpandoracontent/LArHelpers/LArVertexHelper.h"
    #include "larpandoracontent/LArObjects/LArEventTopology.h"

    using namespace pandora;

    namespace lar_content
    {  //remember to close this at the end!

- The Default Settings constructor, which sets default values for any switches you have created.

.. code-block::

   VertexMonitoringAlgorithm::VertexMonitoringAlgorithm() :
    m_visualise{true},
    m_writeFile{false},
    m_useSecondaries{false},
    m_transparencyThresholdE{-1.f},
    m_energyScaleThresholdE{1.f},
    m_scalingFactor{1.f}
    {
    }

- The destructor, which cleans up after everything else is done.

.. code-block::

   VertexMonitoringAlgorithm::~VertexMonitoringAlgorithm()
   {
    if (m_writeFile)
    {
        try
        {
            PANDORA_MONITORING_API(SaveTree(this->GetPandora(), m_treename.c_str(), m_filename.c_str(), "UPDATE"));
        }
        catch (StatusCodeException e)
        {
            std::cout << "VertexMonitoringAlgorithm: Unable to write to ROOT tree" << std::endl;
        }
    }
   }

- The Run loop. This is the main part of the code which will run when the algorithm does and where you can reference other functions to be run from.

.. code-block::

   StatusCode VertexMonitoringAlgorithm::Run()
   {
    if (m_visualise)
    {
        PANDORA_MONITORING_API(SetEveDisplayParameters(
            this->GetPandora(), true, DETECTOR_VIEW_XZ, m_transparencyThresholdE, m_energyScaleThresholdE, m_scalingFactor));
    }

    if (!m_useSecondaries)
        this->AssessVertices();
    else
        this->AssessSecondaryVertices();

    if (m_visualise)
    {
        PANDORA_MONITORING_API(ViewEvent(this->GetPandora()));
    }

    return STATUS_CODE_SUCCESS;
   }

- The ReadSettings. This makes the switches you can edit in the xml - for example if you want it to visualise events or not.

.. code-block::

   StatusCode VertexMonitoringAlgorithm::ReadSettings(const TiXmlHandle xmlHandle)
   {
    PANDORA_RETURN_RESULT_IF_AND_IF(STATUS_CODE_SUCCESS, STATUS_CODE_NOT_FOUND, !=, XmlHelper::ReadValue(xmlHandle, "Visualize", m_visualise));
    PANDORA_RETURN_RESULT_IF_AND_IF(STATUS_CODE_SUCCESS, STATUS_CODE_NOT_FOUND, !=, XmlHelper::ReadValue(xmlHandle, "WriteFile", m_writeFile));
    PANDORA_RETURN_RESULT_IF_AND_IF(STATUS_CODE_SUCCESS, STATUS_CODE_NOT_FOUND, !=, XmlHelper::ReadValue(xmlHandle, "UseSecondaries", m_useSecondaries));

    if (m_writeFile)
    {
        PANDORA_RETURN_RESULT_IF(STATUS_CODE_SUCCESS, !=, XmlHelper::ReadValue(xmlHandle, "Filename", m_filename));
        PANDORA_RETURN_RESULT_IF(STATUS_CODE_SUCCESS, !=, XmlHelper::ReadValue(xmlHandle, "Treename", m_treename));
    }
    if (m_useSecondaries)
    {
        PANDORA_RETURN_RESULT_IF(STATUS_CODE_SUCCESS, !=, XmlHelper::ReadValue(xmlHandle, "SecondaryVertexListName", m_secVertexListName));
    }

    PANDORA_RETURN_RESULT_IF_AND_IF(
        STATUS_CODE_SUCCESS, STATUS_CODE_NOT_FOUND, !=, XmlHelper::ReadValue(xmlHandle, "TransparencyThresholdE", m_transparencyThresholdE));
    PANDORA_RETURN_RESULT_IF_AND_IF(
        STATUS_CODE_SUCCESS, STATUS_CODE_NOT_FOUND, !=, XmlHelper::ReadValue(xmlHandle, "EnergyScaleThresholdE", m_energyScaleThresholdE));
    PANDORA_RETURN_RESULT_IF_AND_IF(STATUS_CODE_SUCCESS, STATUS_CODE_NOT_FOUND, !=, XmlHelper::ReadValue(xmlHandle, "ScalingFactor", m_scalingFactor));

    return STATUS_CODE_SUCCESS;
   }

The key parts of the .h file are:

- Giving it a name:

.. code-block::

   #ifndef LAR_VERTEX_MONITORING_ALGORITHM_H
   #define LAR_VERTEX_MONITORING_ALGORITHM_H 1

   #endif //This goes at the very end!

- You need the namespace again.

.. code-block::

   namespace lar_content
   { //remember to close it at the end!

- Set up your class.

.. code-block::

   class VertexMonitoringAlgorithm : public pandora::Algorithm
   { //remember to close it!

Inside the class, you put your public and private members.

- Public members are accessible generally. The constructor and destructor go in here.

.. code-block::

   public:
    /**
   *  @brief  Default constructor
   */
    VertexMonitoringAlgorithm();

    virtual ~VertexMonitoringAlgorithm();

- Private members are only accessible inside the algorithm. Here’s where you put your switches from ReadSettings, your Run and ReadSettings headers, and the headers for any other functions within your algorithm.

.. code-block::

   private:
    pandora::StatusCode AssessVertices() const;
    pandora::StatusCode AssessSecondaryVertices() const;

    pandora::StatusCode Run();
    pandora::StatusCode ReadSettings(const pandora::TiXmlHandle xmlHandle);

    bool m_visualise;                ///< Whether to produce visual monitoring output
    bool m_writeFile;                ///< Whether to produce ROOT output file
    bool m_useSecondaries;           ///< Whether to assess secondary vertices
    std::string m_filename;          ///< The filename of the ROOT output file
    std::string m_treename;          ///< The name of the ROOT tree
    std::string m_secVertexListName; ///< The name of the list containing secondary vertices
    float m_transparencyThresholdE;  ///< Cell energy for which transparency is saturated (0%, fully opaque)
    float m_energyScaleThresholdE;   ///< Cell energy for which color is at top end of continous color palette
    float m_scalingFactor;           ///< TEve works with [cm], Pandora usually works with [mm] (but LArContent went with cm too)

Once you have created your new algorithm, you can then add it to your Pandora Settings xml file.  It should look something like this:

.. code-block::

   <algorithm type = "LArCutClusterCharacterisation">
        <InputClusterListNames>ClustersU ClustersV ClustersW</InputClusterListNames>
        <OverwriteExistingId>true</OverwriteExistingId>
   </algorithm>

with the algorithm name at the top and the settings switches you defined in the algorithm in the middle. Then build, and run with your custom fcl and xml and you should see your new alg!
